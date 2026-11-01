# 05. Seed Data and Testing

## 1. Seed script (F9)

Command: `python -m app.seed [--employees 10000] [--seed 42] [--reset]`
- Refuses to run if employees already exist, unless `--reset` (drops all data, re-runs the migration, reseeds).
- Deterministic: all randomness comes from one `random.Random(seed)` instance. Names come from `Faker` seeded with the same value (locale per country). "Today" for the seed is the fixed date `2026-10-01`, not the real clock, so two runs give identical data.
- Creates the single HR user from `SEED_HR_EMAIL` / `SEED_HR_PASSWORD`. If the password is empty, generate a random one and print it once. Never store a default password in the repo.
- Target: 10,000 employees and their history in under 15 seconds.

### 1.1 Reference data
- **Currency rates** (illustrative and static, `updated_at` fixed): USD 1.0, INR 83.0, GBP 0.79, EUR 0.92, CAD 1.36, SGD 1.34.
- **Countries:** India (INR), United States (USD), United Kingdom (GBP), Germany (EUR), Canada (CAD), Singapore (SGD).
- **Departments (9):** Executive, Engineering, Product, Sales, Marketing, Customer Support, Finance, Human Resources, Operations.
- **Job titles (level):**
  - Executive: Chief Executive Officer (7)
  - VPs (6): VP of Engineering, VP of Product, VP of Sales, VP of Marketing, VP of Customer Support, VP of Finance, VP of Human Resources, VP of Operations
  - Directors (5): Director of Engineering, Product, Sales, Marketing, Customer Support, Finance, Human Resources, Operations
  - Managers (4): Engineering Manager, Group Product Manager, Sales Manager, Marketing Manager, Support Team Lead, Finance Manager, HR Manager, Operations Manager
  - Individual contributors (1 to 4): Software Engineer (2), Senior Software Engineer (3), Staff Engineer (4), Associate Product Manager (2), Product Manager (3), Senior Product Manager (4), Sales Development Rep (1), Account Executive (2), Senior Account Executive (3), Marketing Associate (1), Marketing Specialist (2), Senior Marketing Specialist (3), Support Specialist (1), Senior Support Specialist (2), Financial Analyst (2), Senior Financial Analyst (3), HR Associate (1), HR Business Partner (3), Operations Associate (1), Operations Analyst (2), Senior Operations Analyst (3)

### 1.2 Distribution

| Item | Value |
|---|---|
| Country mix | India 30%, United States 28%, United Kingdom 12%, Germany 12%, Canada 9%, Singapore 9% (CEO is in the US) |
| Department mix (non-executive) | Engineering 30%, Sales 18%, Customer Support 14%, Operations 10%, Marketing 8%, Product 8%, Finance 7%, Human Resources 5% |
| Inactive | 3% of individual contributors only, so no inactive manager has active reports |

### 1.3 Reporting hierarchy (5 levels, no cycles by construction)

| Tier | Count | Reports to |
|---|---|---|
| 0 CEO | 1 | nobody |
| 1 VPs | 8 | CEO (one per non-executive department) |
| 2 Directors | 80 | A VP of the same department |
| 3 Managers | 700 | A director of the same department |
| 4 Individual contributors | 9,211 | A manager of the same department (spread evenly, about 12 per manager) |

IDs are assigned in tier order, so a manager's id is always lower than their reports'. Insert tier by tier. A report's hire date is never earlier than their manager's.

With `--employees N` other than 10,000, scale tier sizes proportionally (minimum 1 CEO and 1 VP per department) so small test datasets stay valid.

### 1.4 Compensation model (USD, then rounded to whole dollars)
- **Base by level:** L1 45k, L2 65k, L3 90k, L4 120k, L5 160k, L6 220k, L7 350k.
- **Country multiplier:** US 1.00, Singapore 0.85, Canada 0.82, Germany 0.80, United Kingdom 0.78, India 0.32.
- **Department multiplier:** Engineering 1.10, Product 1.05, Sales 1.00, Finance 1.00, Marketing 0.95, Human Resources 0.90, Operations 0.88, Customer Support 0.80, Executive 1.00.
- **Noise:** multiply by a log-normal factor (sigma 0.12).
- **Bonus:** L1 to L2 0 to 5% of base, L3 to L4 5 to 12%, L5 and above 10 to 25%.
- **Allowances:** 2 to 6% of base.
- **Outliers:** about 0.5% of individual contributors get a base multiplied by 1.6 to 2.2, so the outlier table has content.
- All amounts are stored as cents (dollars times 100).

### 1.5 Salary history
For each employee, generate the pay at hire from the model, then a chain of raises:
- Number of raises is limited by tenure (at most one per full year since hire, maximum 4), chosen randomly.
- Each raise is 3 to 10% on base (rounded to whole dollars); bonus and allowances are re-derived with the same percentages as before.
- Dates fall after the hire date and not later than the seed "today", at least 6 months apart.
- The first row is `hire` with reason "New hire" and null previous values; later rows are `manual_edit` with reason "Annual review". The newest row becomes the employee's **current** pay, so the invariant "current = latest history" holds by construction.
- About 25,000 to 30,000 history rows in total.

### 1.6 Performance of the seed
Use SQLAlchemy Core `insert()` with lists of dicts in batches of 1,000 inside one transaction per table; set `PRAGMA synchronous = OFF` and a large cache for the seed connection only. Do not use ORM objects per row.

### 1.7 Seed verification (`python -m app.seed --verify`, also used by tests)
Checks and prints: employee count, history count, country and department mix within 2 percentage points of target, tier counts, no cycles, no inactive employee with active reports, current pay equals latest history for every employee, every employee's manager is active.

## 1a. Test-driven development (how every slice is built)

**Rule: no production code without a failing test that needs it.**

### The loop (repeat in small steps)
1. **Red.** Write one small test for the next behaviour in the spec. Run it. It must fail, and for the right reason (a missing behaviour, not a typo or import error).
2. **Green.** Write the least code that makes it pass. Hard-coding a value is fine if the next test forces the general version.
3. **Refactor.** With everything green, improve names, remove duplication and keep the layering (routes, services, repositories). Run the whole fast suite again.
4. Commit when green.

Use `make test-watch` so each change re-runs the relevant tests in seconds.

### Order inside a slice (outside-in)
- **Backend:** (a) write a short test list from the module's rows in sec 2; (b) service tests first (business rules, in-memory database, fixed `Clock`); (c) API tests for the route (status codes, auth, CSRF, validation, response shape); (d) implement service, then repository, then route; (e) run `make openapi`.
- **Frontend:** (a) add MSW handlers for the endpoint from the API contract (`docs/openapi.json`); (b) write page and component tests for what the user sees and does (loading, empty, error, validation, URL state); (c) implement the page; (d) refactor.
- Then smoke test the feature in the browser on the seeded database, before moving to the next slice.

### What is tested first, and what is not
| Test first | No test needed |
|---|---|
| Business rules, validation, calculations (money, rounding, bulk update math, salary bands) | Pure styling and theme tokens |
| Queries: filters, sorting, pagination, search | Static layout markup without behaviour |
| Auth and security rules | Config files, docs |
| UI behaviour: loading, empty and error states, form validation, URL state, confirmations | |
| Every bug fix: write a failing test that reproduces the bug first | |

### Test quality rules
- Test behaviour, not implementation. Test names describe the behaviour (`test_deactivate_is_blocked_when_employee_has_active_reports`).
- One reason to fail per test. Arrange, act, assert.
- Use a real in-memory SQLite database; do not mock repositories. Fake only time (`Clock`) and, in the browser, the network (MSW).
- Build data with factories. No shared mutable state, no sleeps, no random values without a seed.
- Keep the suite fast; mark slow tests (`slow`) and run them separately.

### Showing TDD in the commit history
- Every commit message has a body line `Tests first: <the tests that were written before the code>`.
- For the riskiest rules (session auth and rate limit, hierarchy cycle detection, bulk update math, band edges), make two commits: a `test(...)` commit with the new tests, then the `feat(...)` commit that makes them pass. To keep CI green in the first commit, mark the new tests as expected failures (`@pytest.mark.xfail(strict=True, reason="red")` in pytest, `it.fails(...)` in Vitest) and remove the marker in the `feat` commit. With `strict=True`, a test that unexpectedly passes fails the run, which forces the marker to be removed.
- Example:
  ```
  feat(F7): bulk update preview and apply API (spec 02 sec 5.5, sec 7.3)

  Tests first: percent rounding half away from zero; fixed amount adds exactly;
  only the chosen component changes; unchanged employees excluded;
  stale preview rolls back; one shared batch_id.
  ```

## 2. Backend testing strategy

**Principles:** fast (in-memory SQLite, no network), deterministic (fixed clock, fixed seeds, no sleeps), readable (one behaviour per test, names like `test_deactivate_is_blocked_when_employee_has_active_reports`).

**Layers**
- *Unit tests (`tests/unit`)* call services directly with an in-memory database and factories. This is where most tests live.
- *API tests (`tests/api`)* use `TestClient` with the real app and an in-memory database to verify routing, auth, CSRF, validation and response shapes.
- *Slow tests* (marker `slow`, excluded from the default run, included in CI and before release) use the real 10k seed.

**Fixtures (`conftest.py`, `factories.py`)**
- `db` fixture: fresh in-memory database per test, schema from the migration, PRAGMAs applied.
- `clock` fixture: fixed `Clock` (for example 2026-10-31).
- `make_employee(**overrides)`, `make_hr_user()`, `make_department()` and so on, with sensible defaults; helpers to build small hierarchies.
- `client` and `auth_client` fixtures; `auth_client` logs in and sends the CSRF header automatically.

**Test plan by module**

| Module | Tests |
|---|---|
| Schema | CHECK constraints reject negative amounts and self-manager; email unique case-insensitively; generated `total_comp_cents` equals the sum; migration schema equals `create_all` schema |
| Auth | login success; wrong password and unknown email return the same message; inactive user cannot log in; rate limit blocks after N failures and resets on success; session expires (absolute and idle) with a fake clock; logout invalidates the token; missing or wrong CSRF returns 403; stored session id is a hash; cookie flags present |
| Employee create | writes employee and one `hire` history row; duplicate email 409; unknown lookup 422; manager must be active; hire date in the future 422; extra fields rejected |
| Employee update | pay change writes one history row with correct previous values; non-pay change writes none; no-op patch writes none; email conflict 409; explicit null manager removes the manager |
| Hierarchy | self, direct and indirect cycles rejected; deactivate blocked with active reports; allowed after reports are reassigned; reactivate blocked when manager inactive; direct reports pagination |
| List | each filter alone and combined; search escapes `%` and `_`; default status active; sort whitelist (bad value 422); stable pagination (no duplicates across pages); `page_size` over 100 returns 422; `total` correct |
| History | newest first; `previous` null for hire; totals correct; invariant test: after a random sequence of creates, edits and bulk updates, current pay equals the latest history row for every employee |
| Bulk update | percent mode: positive and negative percentages, rounding half away from zero; fixed-amount mode: adds exactly the amount; each component (base, bonus, allowances) changes only its own column and leaves the other two untouched; percent change of a zero bonus excludes that employee (no history row, not counted); only active employees; empty filter without `apply_to_all` rejected; wrong mode/field combination, zero or out-of-range percentage, non-positive amount and unknown component return 422; a result above the maximum returns 422 `result_too_large` and changes nothing; apply writes one history row per affected employee with one shared `batch_id` and correct previous values; preview sums equal the applied sums; `stale_preview` changes nothing (rollback verified); minimum base of 1 cent |
| History search | each filter alone and combined (`employee` by name and by email, `reason`, `change_type`, date range inclusive on both ends, `batch_id`); `%` and `_` in search text treated literally; sort whitelist (bad value 422); `date_from` after `date_to` returns 422; stable pagination; inactive employees' history included; total correct; no per-row queries; 401 without a session |
| Currency rates | create valid; duplicate 409; lowercase or wrong-length code 422; rate zero, negative, too large or not finite 422; update changes the rate and `updated_at` (fixed clock); updating or deleting USD rejected; delete of an unused currency succeeds; delete of a currency used by a country returns 409 naming the countries; unknown code 404; changed rate shows in `GET /api/lookups` and in the employee's local amounts while stored USD is unchanged; 401 without session; missing CSRF 403 |
| Analytics | golden dataset tests in `03` sec 5 |
| Currency | local conversion uses the country's rate; rounding half up; missing rate impossible by FK |
| Security | parametrized test: every route except health and login returns 401 without a session; response bodies never contain `password_hash`; security headers present; `sort_by` injection attempts return 422 |
| Errors | each error code returns the standard shape; 500 does not leak a stack trace |
| Query plans | `EXPLAIN QUERY PLAN` assertions from `01` sec 8 on the seeded database |
| Seed | small seed (500) is deterministic (checksum equal across two runs), passes `--verify`; slow test with the full 10k also passes `--verify` within the time target |
| Perf smoke (slow) | on the 10k seed: list, detail, each analytics endpoint under their targets (use generous thresholds, for example 3x the target, to avoid flaky failures) |

**Coverage:** at least 85% line coverage overall and 90% for `services/`, enforced by `pytest --cov --cov-fail-under`. Coverage is a floor, not the goal; each business rule above must have a test that would fail if the rule were removed.

## 3. Frontend testing
See `04-frontend-spec.md` sec 8. Same principles: no network, no real timers, one behaviour per test.

## 4. Quality gates and CI (optional but recommended)
A GitHub Actions workflow runs on push: backend (ruff, pytest with coverage, slow tests), frontend (eslint, vitest, build). Local equivalent: `make lint test`.

## 5. AI usage artifacts
- Save every prompt and the key parts of responses in `prompts/YYYY-MM-DD-NN-topic.md` (raw, not edited). Start with the PRD conversation and these specs.
- In `docs/DESIGN.md` add a short "How AI was used" section: what was generated, what was reviewed by hand, and one or two cases where the AI output was wrong or was changed and why.
- Each commit message names the spec it implements, for example `backend: bulk update preview and apply (spec 02 sec 5.5, sec 7.3)`.
