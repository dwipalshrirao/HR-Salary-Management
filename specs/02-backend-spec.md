# 02. Backend Specification (FastAPI)

Depends on: `00-overview.md` (conventions), `01-database-schema.md` (tables).
Analytics endpoints are specified in `03-analytics-spec.md`.

## 1. Stack and layout
Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.x (sync), Alembic, argon2-cffi, pytest. Layout is in `00-overview.md` sec 5. `python -m app.export_openapi` writes the API contract to `docs/openapi.json` (run through `make openapi` after any API change); the frontend tests and MSW handlers are written from it.

Layer rules:
- `api/routes/*`: parse input (Pydantic), call a service, return a schema. No SQL, no business rules.
- `services/*`: business rules, validation that needs the database, transactions. Receive a `Session` and a `Clock`.
- `repositories/*`: SQLAlchemy queries only. Return rows or model objects.
- `schemas/*`: Pydantic request/response models (these generate the OpenAPI used by the frontend).

## 2. Configuration
`app/config.py` uses `pydantic-settings` to read the variables in `00-overview.md` sec 6. Settings are created once and injected via a dependency so tests can override them.

## 3. Database session and transactions
- `db.py` creates the engine, registers the PRAGMA listener (`01` sec 7) and exposes `get_db()`.
- One request uses one `Session`. Write endpoints commit once at the end of the service call; any exception rolls back.
- `clock.py` exposes `Clock.today() -> date` and `Clock.now() -> datetime (UTC)`. A `get_clock()` dependency returns the system clock; tests override it with a fixed clock.

## 4. Authentication and sessions (F1)

**Model.** Server-side sessions. The cookie holds a random token; the database holds only its SHA-256 hash.

**Login** `POST /api/auth/login`
1. Require `Content-Type: application/json`.
2. Check the rate limiter for `(client_ip, lower(email))`: after `LOGIN_MAX_FAILURES` failures within `LOGIN_WINDOW_MINUTES`, return 429 with `Retry-After`. (In-memory, single process; documented limitation.)
3. Look up the active user by email. Verify the password with argon2id. To avoid revealing which part failed, a wrong email and a wrong password return the same 401 `Invalid email or password`. Run a dummy hash verify when the user does not exist to keep timing similar.
4. On success: create a session with `secrets.token_urlsafe(32)` (store `sha256(token)` as `id_hash`), a new `csrf_token`, `expires_at = now + SESSION_ABSOLUTE_HOURS`. Always issue a new session at login (prevents session fixation). Reset the failure counter.
5. Set cookie `SESSION_COOKIE_NAME`: `HttpOnly`, `SameSite=Lax`, `Path=/`, `Secure` when `COOKIE_SECURE=true`, `Max-Age` equal to the absolute lifetime.
6. Return `200 { "user": {id, email, full_name}, "csrf_token": "..." }`.

**Current user** `GET /api/auth/me`: returns the same body when the session is valid, otherwise 401.

**Logout** `POST /api/auth/logout` (CSRF required): delete the session row, clear the cookie, return 204.

**Session validation** (dependency `current_user`, used by every protected route):
- Hash the cookie token, load the session and user. Reject (401) if missing, expired (`expires_at`), idle longer than `SESSION_IDLE_MINUTES` (`last_seen_at`), or the user is inactive.
- Update `last_seen_at` at most once per minute to avoid a write on every request.

**CSRF** (dependency `csrf_protect`, applied to every `POST`, `PUT`, `PATCH`, `DELETE` except login): require header `X-CSRF-Token` equal to the session's `csrf_token` (compare with `secrets.compare_digest`), otherwise 403 `csrf_failed`. The login endpoint cannot have a token yet; it relies on `SameSite=Lax` plus the JSON content-type requirement.

**Cleanup.** Delete expired sessions on login (cheap, uses `idx_sessions_expires`).

## 5. API contract

All routes are under `/api`. All except `/api/health` and `POST /api/auth/login` require a valid session. All responses are JSON and carry `Cache-Control: no-store`.

### 5.1 Conventions
Pagination, errors and money are defined in `00-overview.md` sec 6. Unknown query parameters are ignored; invalid values return 422.

### 5.2 Auth, health, lookups

| Method and path | Purpose | Success |
|---|---|---|
| `POST /api/auth/login` | Sign in | 200 |
| `POST /api/auth/logout` | Sign out | 204 |
| `GET /api/auth/me` | Current user and CSRF token | 200 |
| `GET /api/health` | Liveness (no auth) | 200 `{"status":"ok"}` |
| `GET /api/lookups` | Reference data | 200 |

`GET /api/lookups` response (cache on the client; `Cache-Control: private, max-age=60`):
```json
{
  "countries":   [{ "code": "IN", "name": "India", "currency_code": "INR" }],
  "departments": [{ "id": 1, "name": "Engineering" }],
  "job_titles":  [{ "id": 1, "name": "Software Engineer", "level": 2 }],
  "currency_rates": [{ "currency_code": "INR", "rate_per_usd": 83.0, "updated_at": "2026-01-01T00:00:00Z" }]
}
```

### 5.3 Employees

| Method and path | Purpose | Success |
|---|---|---|
| `GET /api/employees` | List with filters, sort, pagination | 200 |
| `POST /api/employees` | Create (writes `hire` history row) | 201 |
| `GET /api/employees/{id}` | Detail | 200 |
| `PATCH /api/employees/{id}` | Partial update (writes history if pay changed) | 200 |
| `POST /api/employees/{id}/deactivate` | Soft delete | 200 |
| `POST /api/employees/{id}/reactivate` | Restore | 200 |
| `GET /api/employees/{id}/history` | Salary history, newest first | 200 |
| `GET /api/employees/{id}/direct-reports` | Paginated direct reports | 200 |

**List query parameters**

| Param | Type | Notes |
|---|---|---|
| `q` | string, max 100 | Matches name or email (case-insensitive contains) |
| `country_code` | string | Exact |
| `department_id` | int | Exact |
| `job_title_id` | int | Exact |
| `manager_id` | int | Direct reports of this manager |
| `status` | `active` (default), `inactive`, `all` | |
| `sort_by` | `full_name` (default), `email`, `hire_date`, `base_salary`, `total_comp`, `department`, `job_title`, `country` | Whitelist mapped to columns in code; never pass user text into `ORDER BY` |
| `sort_dir` | `asc` (default), `desc` | Secondary sort is always `id` for stable pages |
| `page`, `page_size` | int | See conventions |

**List item**
```json
{
  "id": 42, "full_name": "Asha Patel", "email": "asha.patel.42@acme.example",
  "department": {"id": 1, "name": "Engineering"},
  "job_title": {"id": 3, "name": "Senior Software Engineer"},
  "country": {"code": "IN", "name": "India"},
  "manager": {"id": 7, "full_name": "Ravi Shah"},
  "hire_date": "2021-04-12", "status": "active",
  "base_salary_cents": 4500000, "bonus_cents": 450000, "allowances_cents": 180000,
  "total_comp_cents": 5130000
}
```
`manager` is `null` for the top of the hierarchy. Build the list with **one joined query plus one count query** (no per-row queries).

**Detail** adds `direct_reports_count` and `local` (see 5.4), plus `created_at`, `updated_at`.

**Create body**
```json
{ "full_name": "...", "email": "...", "department_id": 1, "job_title_id": 3, "country_code": "IN",
  "manager_id": 7, "hire_date": "2026-10-01",
  "base_salary_cents": 4500000, "bonus_cents": 0, "allowances_cents": 0 }
```

**Patch body**: any subset of `full_name, email, department_id, job_title_id, country_code, manager_id (nullable), hire_date, base_salary_cents, bonus_cents, allowances_cents`, plus optional `change_reason` (max 200, used only when pay changes; default "Manual edit"). Use `model_fields_set` so an explicit `"manager_id": null` (remove manager) differs from an omitted field. `status` cannot be patched.

**History item**
```json
{ "id": 9, "effective_date": "2026-10-31", "change_type": "bulk_update", "reason": "FY27 raise",
  "batch_id": "6f1c...", "created_at": "...",
  "new":      { "base_salary_cents": 4725000, "bonus_cents": 450000, "allowances_cents": 180000, "total_comp_cents": 5355000 },
  "previous": { "base_salary_cents": 4500000, "bonus_cents": 450000, "allowances_cents": 180000, "total_comp_cents": 5130000 } }
```
`previous` is `null` for `hire`. Totals are computed in the service.

### 5.4 Local currency (display only)
Employee detail includes:
```json
"local": { "currency_code": "INR", "rate_per_usd": 83.0,
           "base_salary_cents": 373500000, "bonus_cents": 37350000,
           "allowances_cents": 14940000, "total_comp_cents": 425790000 }
```
Amounts are in local minor units: `round(usd_cents * rate_per_usd)` using round-half-up on the product. The country's currency comes from `countries.currency_code`. These values are never stored and never used in analytics.

### 5.5 Bulk update (F7)

| Method and path | Purpose |
|---|---|
| `POST /api/bulk-updates/preview` | Calculate the effect; changes nothing |
| `POST /api/bulk-updates/apply` | Apply in one transaction |

Request (both). Example: add $1,200 per year to the allowances of one department in one country.
```json
{ "filter": { "country_code": "IN", "department_id": 1, "job_title_id": null, "manager_id": null },
  "apply_to_all": false,
  "component": "allowances",
  "mode": "fixed_amount",
  "percentage": null,
  "amount_cents": 120000 }
```
`apply` also requires `"expected_count": 123` and `"reason": "FY27 team allowance"` (1 to 200 chars).

Rules:
- Only **active** employees are matched. Each request changes **one component**: `component` is `base`, `bonus` or `allowances`. The other two components stay as they are.
- `mode = "percent"`: `percentage` is required (decimal with at most 2 places, between `BULK_MIN_PERCENT` and `BULK_MAX_PERCENT`, not 0) and `amount_cents` must be absent. Convert to integer basis points (`bp = round(percentage * 100)`) before any math. New value = `CAST(ROUND(current * (10000 + bp) / 10000.0) AS INTEGER)`, rounding half away from zero (SQLite `ROUND`).
- `mode = "fixed_amount"`: `amount_cents` is required (integer from 1 to `MAX_AMOUNT_CENTS`, annual, USD, always **added**) and `percentage` must be absent. New value = `current + amount_cents`. To reduce a component use `percent` mode with a negative percentage.
- A request that sends the wrong combination (for example `percent` mode with `amount_cents`) returns 422.
- All amounts are **annual**. A monthly amount is entered as 12 times its value (for example $100 per month is `120000`). A bulk update changes the current annual amount **immediately**. One-off payments, month-specific payments and future-dated changes are not modelled.
- **Affected employees** are matched employees whose value would actually change. For example, a percentage change of a bonus that is 0 changes nothing, so those employees are not counted and get no history row.
- A request with an empty filter and `apply_to_all=false` is rejected (422). This protects against an accidental organisation-wide change.
- If any resulting value would exceed `MAX_AMOUNT_CENTS`, reject the request with 422 `result_too_large` and change nothing. A base salary never goes below 1 cent.

Preview response:
```json
{ "component": "allowances", "mode": "fixed_amount", "affected_count": 123,
  "sum_before_cents": 0, "sum_after_cents": 0, "sum_delta_cents": 0,
  "sample": [ { "id": 42, "full_name": "...", "before_cents": 0, "after_cents": 0 } ] }
```
The sums are totals of the chosen component over the affected employees. `sample` has up to 10 employees, ordered by id. If `affected_count` is 0, return 200 with zeros and an empty sample.

Apply response: `{ "batch_id": "...", "component": "allowances", "affected_count": 123, "sum_delta_cents": 0 }`. If the number of affected employees at apply time differs from `expected_count`, change nothing and return 409 `stale_preview`.

### 5.6 Analytics
See `03-analytics-spec.md`. All under `/api/analytics/...`.

### 5.7 Salary history search (F10)

| Method and path | Purpose | Success |
|---|---|---|
| `GET /api/salary-history` | Every pay change across all employees, with search and filters | 200 |

Query parameters:

| Param | Type | Notes |
|---|---|---|
| `employee` | string, max 100 | Case-insensitive contains match on the employee's full name or email |
| `reason` | string, max 100 | Case-insensitive contains match on the reason |
| `change_type` | `hire`, `manual_edit`, `bulk_update` | Exact |
| `date_from`, `date_to` | `YYYY-MM-DD` | Inclusive range on `effective_date`; `date_from` later than `date_to` returns 422 |
| `batch_id` | string | Exact; used to show every row of one bulk update |
| `sort_by` | `effective_date` (default), `employee`, `change_type` | Whitelist mapped to columns in code |
| `sort_dir` | `desc` (default), `asc` | Ties break by history `id` in the same direction |
| `page`, `page_size` | int | See conventions |

Item (the same `new` and `previous` blocks as the per-employee history in 5.3, plus the employee):
```json
{ "id": 9,
  "employee": { "id": 42, "full_name": "Asha Patel", "email": "asha.patel.42@acme.example" },
  "effective_date": "2026-10-31", "change_type": "bulk_update", "reason": "FY27 raise", "batch_id": "6f1c...",
  "new":      { "base_salary_cents": 4725000, "bonus_cents": 450000, "allowances_cents": 180000, "total_comp_cents": 5355000 },
  "previous": { "base_salary_cents": 4500000, "bonus_cents": 450000, "allowances_cents": 180000, "total_comp_cents": 5130000 } }
```
Notes:
- History of inactive employees is included.
- The acting HR user is not recorded and not returned (audit trail is out of scope).
- Build the page with **one joined select plus one count query** (no per-row queries). Escape `%` and `_` in search text and use `LIKE ... ESCAPE`.

### 5.8 Currency rates (F11, API only, no screen)

HR does not need to change rates in the app, so there is no UI for this. The endpoints exist so rates can be maintained through the API (for example with `curl` or the generated OpenAPI docs). They use the same session and CSRF rules as every other route.

| Method and path | Purpose | Success |
|---|---|---|
| `GET /api/currency-rates` | List all rates, sorted by code | 200 |
| `GET /api/currency-rates/{code}` | One rate | 200 |
| `POST /api/currency-rates` | Create a currency | 201 |
| `PATCH /api/currency-rates/{code}` | Change the rate | 200 |
| `DELETE /api/currency-rates/{code}` | Delete a currency | 204 |

Item:
```json
{ "currency_code": "EUR", "rate_per_usd": 0.92, "updated_at": "2026-11-01T09:30:00Z", "country_codes": ["DE"] }
```
`country_codes` lists the countries that use the currency.

Create body: `{ "currency_code": "EUR", "rate_per_usd": 0.92 }`. Patch body: `{ "rate_per_usd": 0.93 }` (the code cannot change, because countries refer to it).

Rules:
- `currency_code` is exactly three uppercase letters (`^[A-Z]{3}$`); anything else returns 422. A code that already exists returns 409.
- `rate_per_usd` means local units per 1 USD. It must be a finite number greater than 0 and at most 1,000,000.
- **USD is the base currency** (all stored amounts are USD): its rate is fixed at 1 and cannot be changed (422) or deleted (409).
- A currency that any country uses cannot be deleted (409, the message lists the country codes).
- Updating a rate sets `updated_at` from the `Clock`. It changes only the local-currency amounts shown from then on; stored USD amounts are never touched.
- Rate changes are not kept in a history (audit trail is out of scope); only `updated_at` is stored.
- `GET /api/lookups` returns the current rates, so the app shows a changed rate after the lookup cache expires (up to one minute) or on reload.
- Countries are seeded reference data. A new currency only has an effect once a country refers to it, and there is no endpoint to add countries.

Example (no screen, so use the API directly):
```
# 1. Sign in; the cookie jar keeps the session, the response contains csrf_token
curl -c jar.txt -H "Content-Type: application/json" \
     -d '{"email":"<hr email>","password":"<password>"}' https://<host>/api/auth/login
# 2. Change a rate, sending the cookie and the CSRF token from step 1
curl -b jar.txt -X PATCH -H "Content-Type: application/json" -H "X-CSRF-Token: <csrf_token>" \
     -d '{"rate_per_usd": 0.93}' https://<host>/api/currency-rates/EUR
```

## 6. Validation rules (Pydantic)

| Field | Rule |
|---|---|
| `full_name` | 1 to 120 chars after trimming |
| `email` | Valid email, max 254, stored as given, compared case-insensitively |
| `*_cents` | Integers; `base_salary_cents` 1 to `MAX_AMOUNT_CENTS`; bonus and allowances 0 to `MAX_AMOUNT_CENTS` |
| `hire_date` | Valid date, not later than today |
| `country_code` | Exists in `countries` |
| `department_id`, `job_title_id` | Exist |
| `manager_id` | Exists, active, not the employee itself, no cycle (see 7.2) |
| `change_reason`, `reason` | Trimmed, max 200 |
| `percentage` | See 5.5 |
| `q` | Max 100 chars; `%` and `_` are escaped before use in `LIKE` |
| `employee`, `reason` (history search) | Max 100 chars each; escaped like `q` |
| `date_from`, `date_to` | Valid dates; `date_from` not later than `date_to` |
| `currency_code` | Exactly three uppercase letters |
| `rate_per_usd` | Finite number greater than 0, at most 1,000,000 |

Cross-field and database-backed rules are checked in services and return 422 (invalid value) or 409 (conflicts with current state, such as a duplicate email).

## 7. Services

### 7.1 EmployeeService
- **create(data)**: validate lookups and manager; insert employee; insert a `hire` history row with `effective_date = hire_date`, reason "New hire" and `prev_*` null; commit once. Duplicate email returns 409 (catch the unique violation).
- **update(id, patch)**: load or 404; validate any changed lookup or manager; apply; if any of the three pay fields changed, insert one `manual_edit` history row (`effective_date = today`, previous values from before the change); set `updated_at`; commit once. A patch that changes nothing returns the current employee and writes nothing.
- **deactivate(id)**: 409 if the employee has active direct reports (message names the count); set `status='inactive'`.
- **reactivate(id)**: 409 if the manager exists and is inactive; set `status='active'`.
- **list(filters, sort, page)** and **get(id)** as in 5.3.
- **history(id)**: ordered by `effective_date DESC, id DESC`; build `new` and `previous` blocks.

### 7.2 HierarchyService
- `validate_manager(employee_id | None, manager_id)`: manager exists, active, not self, no cycle (query in `01` sec 6). Used by create (no employee id yet, so only existence and active checks) and update.
- `direct_reports(manager_id, status, page)` and `active_report_count(manager_id)`.

### 7.3 BulkUpdateService
- **Component and expression.** Map `component` to a column through a whitelist (`base` to `base_salary_cents`, `bonus` to `bonus_cents`, `allowances` to `allowances_cents`); never use client text as a column name. Build the new-value expression from `mode`: percent gives `CAST(ROUND(col * (10000 + :bp) / 10000.0) AS INTEGER)`, fixed amount gives `col + :amount`.
- **Matching set.** The filter (shared builder with the list query), `status = 'active'`, and `new_value <> col` (so unchanged rows are excluded).
- **preview(request)**: one aggregate query over the matching set returning `COUNT(*)`, `SUM(col)`, `SUM(new_value)` and `MAX(new_value)`, plus one limited select for the sample. Reject with `result_too_large` when `MAX(new_value)` exceeds `MAX_AMOUNT_CENTS`.
- **apply(request, expected_count, reason)**, in one transaction:
  1. Run the aggregate; if the count differs from `expected_count`, roll back and raise `stale_preview`; if `MAX(new_value)` is too large, roll back and raise `result_too_large`.
  2. Generate `batch_id = uuid4()`.
  3. `INSERT INTO salary_history (...) SELECT ... FROM employees WHERE <matching set>` writing the new value for the chosen component, the current values for the other two, the **previous** values for all three, `change_type='bulk_update'`, `effective_date=today`, `batch_id`, `reason`.
  4. `UPDATE employees SET <col> = <new_value>, updated_at = :now WHERE <matching set>`. Run this after the insert, using the same matching set, which is still valid because the insert did not change `employees`.
  5. Verify the updated row count equals the inserted history count and `expected_count`; otherwise roll back.
  Both statements are set-based, so 10,000 rows takes milliseconds. Do not loop over employees in Python.

### 7.4 HistoryService (F10)
- **search(filters, sort, page)**: one select joining `salary_history h` to `employees e`, with these conditions when present: `(e.full_name LIKE :p ESCAPE '\' OR e.email LIKE :p ESCAPE '\')` for `employee`, `h.reason LIKE :r ESCAPE '\'` for `reason`, `h.change_type = :t`, `h.effective_date BETWEEN :from AND :to`, `h.batch_id = :b`. Search text is escaped and wrapped as `%text%`. `sort_by` maps through a whitelist (`effective_date` to `h.effective_date`, `employee` to `e.full_name`, `change_type` to `h.change_type`) with `h.id` as the tie-breaker. A second query counts the same filter for `total`.
- Build the `new` and `previous` blocks with the same helper the per-employee history uses (totals computed in code, no second query).

### 7.5 CurrencyRateService (F11)
- **list()**: all rows ordered by code, with `country_codes` from one grouped query over `countries`.
- **get(code)**: 404 if missing.
- **create(code, rate)**: validate format and range; 409 if the code exists; `updated_at` from the `Clock`.
- **update_rate(code, rate)**: 404 if missing; 422 for `USD`; validate range; set `rate_per_usd` and `updated_at`.
- **delete(code)**: 404 if missing; 409 for `USD`; 409 if any country uses it (message lists the codes).
- Routes are thin: parse, call the service, return the item.

## 8. Error handling
- A global exception handler converts `AppError` subclasses (`NotFoundError`, `ConflictError`, `StaleCountError`, `RateLimitedError`, `UnauthorizedError`, `CsrfError`) and Pydantic validation errors into the shape in `00-overview.md` sec 6. Validation `details` list `{field, message}` pairs.
- Unexpected exceptions return 500 `internal_error` with a request id and a generic message. Stack traces go to logs only.

## 9. Security checklist (each item has a test)
- Every route except health and login requires a session (a parametrized test walks the router).
- State-changing routes require a valid CSRF header.
- Passwords hashed with argon2id; session tokens stored hashed; neither is ever logged or returned.
- Login: generic error, rate limit, new session per login.
- Cookie flags: `HttpOnly`, `SameSite=Lax`, `Secure` when enabled.
- SQL: ORM or bound parameters only; `sort_by` is a whitelist; `LIKE` input is escaped.
- Pydantic models forbid extra fields on write bodies (`extra="forbid"`), so clients cannot set `status`, ids or totals.
- Responses never include `password_hash` or session data.
- Security headers on every response: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store` on API routes.
- CORS disabled by default (same-origin). If a `CORS_ORIGINS` list is set, use exact origins with credentials, never `*`.
- Secrets only from the environment; `.env` is gitignored.

## 10. Performance
- List: one joined select with `LIMIT/OFFSET` plus one `COUNT(*)` on the same filter. Page size capped at 100.
- Avoid N+1: join department, title, country and manager in the list query.
- Bulk update and seed use set-based SQL / batched inserts.
- Measure on the seeded database with the perf smoke script (`05` sec 2) before declaring the list target met.

## 11. Logging
Request-id middleware; one structured log line per request (method, path, status, duration ms, user id). No bodies, no amounts, no tokens.
