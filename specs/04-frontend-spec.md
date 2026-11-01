# 04. Frontend Specification (React + Vite + JavaScript)

Depends on: `00-overview.md`, `02-backend-spec.md` (API contract), `03-analytics-spec.md`.

## 1. Stack
React 18 and Vite, written in **JavaScript** (ES modules, `.jsx` for files that contain JSX; no TypeScript), MUI + MUI X DataGrid (community), TanStack Query, React Router, React Hook Form + Zod, Recharts, Vitest + React Testing Library + MSW. Prettier and ESLint with the React, React hooks and jsx-a11y rules.

## 2. Project structure
```
frontend/src/
  api/          client.js, auth.js, employees.js, bulk.js, history.js, analytics.js, lookups.js
  auth/         AuthProvider.jsx, ProtectedRoute.jsx, useAuth.js
  components/   AppShell, Money, StatCard, ConfirmDialog, EmptyState, ErrorState, PageHeader,
                FilterBar, EmployeeAutocomplete, StatusChip, charts/{BarChartCard,DistributionChart}
  pages/        LoginPage, DashboardPage, EmployeesPage, EmployeeDetailPage, EmployeeFormPage,
                HistoryPage, BulkUpdatePage, NotFoundPage
  hooks/        useDebounce, useUrlState, useLookups
  lib/          money.js, format.js, schemas.js (zod)
  theme.js  router.jsx  main.jsx
  test/         setup.js, server.js (MSW), handlers.js, render.jsx
```
The API contract is `docs/openapi.json`, exported from the backend with `make openapi`; do not edit it by hand. The frontend is plain JavaScript, so there are no generated TypeScript types: write the API functions, MSW handlers and tests from the contract, and describe shared data shapes with JSDoc `@typedef` comments. Field names stay snake_case.

## 3. Dev setup and routing
- In development the Vite dev server proxies `/api` to the API on port 8000. The browser sees one origin, so cookies work and CORS is not needed. When deployed, the built SPA and the API are served from one origin (reverse proxy or static mount), so the same cookie and CSRF rules apply. Fetch uses `credentials: "same-origin"`.
- Routes:

| Path | Page | Guard |
|---|---|---|
| `/login` | LoginPage | Redirects to `/` if already signed in |
| `/` | DashboardPage | Protected |
| `/employees` | EmployeesPage | Protected |
| `/employees/new` | EmployeeFormPage (create) | Protected |
| `/employees/:id` | EmployeeDetailPage | Protected |
| `/employees/:id/edit` | EmployeeFormPage (edit) | Protected |
| `/history` | HistoryPage | Protected |
| `/bulk-update` | BulkUpdatePage | Protected |
| `*` | NotFoundPage | |

`ProtectedRoute` waits for the initial `GET /api/auth/me`, shows a spinner while loading, and redirects to `/login` (remembering the original path) when signed out.

## 4. API client (`api/client.js`)
- One `request<T>(method, path, { query, body })` function built on `fetch`.
- Sends JSON, parses JSON, and for non-2xx responses throws `ApiError { status, code, message, details }` from the standard error shape.
- Adds `X-CSRF-Token` (kept in memory in the auth store, never in `localStorage`) on `POST/PUT/PATCH/DELETE`.
- On `401` from any call other than login: clear the auth state and navigate to `/login`. This is handled once, here, not per screen.
- Typed wrapper functions per resource (`listEmployees(params)`, `createEmployee(body)`, and so on).

## 5. State, data fetching and formatting

**Query keys and invalidation**

| Key | Fetched by | Invalidated by |
|---|---|---|
| `["lookups"]` | all forms and filters (`staleTime: Infinity`) | never |
| `["employees", params]` | list | create, update, deactivate, reactivate, bulk apply |
| `["employee", id]` | detail | update, deactivate, reactivate, bulk apply |
| `["employee", id, "history"]` | history | update, bulk apply |
| `["history", params]` | History page | create, update, bulk apply |
| `["employee", id, "reports", page]` | direct reports | any employee mutation |
| `["analytics", ...]` | dashboard (`staleTime: 60s`) | create, update, deactivate, reactivate, bulk apply |

Tables use `placeholderData: keepPreviousData` so pagination and sorting do not flash empty.

**Money (`lib/money.js`)**
- `formatUsd(cents)` returns e.g. `$45,000` (no decimals for whole dollars, two decimals otherwise) using `Intl.NumberFormat("en-US")`.
- `formatUsdCompact(cents)` returns e.g. `$812.3M` or `$83.7K` (compact notation, one decimal) for KPI cards.
- `formatLocal(cents, currencyCode)` uses `Intl.NumberFormat("en-US", { style: "currency", currency, maximumFractionDigits: 0 })`, giving e.g. `₹4,257,900`.
- `parseDollarsToCents(input: string)`: string-based parsing (split on `.`), never `parseFloat * 100`. Rejects more than 2 decimals, negatives and non-numbers.
- `centsToDollarsString(cents)` for filling form inputs.
- Unit tests cover `0.1 + 0.2`-style cases, large values, compact formatting and invalid input.

**Dates** are shown with `Intl.DateTimeFormat` (`31 Oct 2026`); the API format stays `YYYY-MM-DD`.

**List page state lives in the URL** (`useUrlState`): `q`, `country_code`, `department_id`, `job_title_id`, `manager_id`, `status`, `sort_by`, `sort_dir`, `page`, `page_size`. This makes views shareable and the browser back button work. Changing any filter resets `page` to 1. Search input is debounced 300 ms.

## 6. Pages

General rules for every page: show a loading skeleton, an error state with a Retry button (from `ApiError.message`), and an empty state with guidance. Layout is desktop first (works from 1024 px) and degrades gracefully to tablet width. All inputs have labels; dialogs trap focus; charts have a text alternative (a data table or `aria-label` summary).

### 6.1 LoginPage
- Fields: email, password. Submit button disabled while the request runs.
- Errors: 401 shows "Invalid email or password"; 429 shows "Too many attempts. Try again in a few minutes."; network error shows a generic message.
- On success store `user` and `csrf_token` in the auth context and go to the originally requested path or `/`.
- Acceptance: pressing Enter submits; password is never shown in the URL or logs; focus goes to the first field on load.

### 6.2 AppShell
Top bar with product name, navigation links (Dashboard, Employees, History, Bulk update), and a user menu with the signed-in name and Logout. Logout calls the API, clears query cache and auth state, and goes to `/login`.

### 6.3 DashboardPage (F8)
Purpose: answer the interviewer's pay questions at a glance. Data from `03-analytics-spec.md`.

Layout, top to bottom:
1. **Controls row:** measure toggle (Total compensation | Base salary), country select, department select ("All" by default). Controls drive every widget below via URL params.
2. **KPI cards (4):** (money in compact format, e.g. `$812.3M`) Total compensation expenditure; Headcount; Average compensation; Highest-paying country (name and its average). Each card shows a skeleton while loading.
3. **Pay mix:** a single horizontal stacked bar for base / bonus / allowances share (from `summary.components`), with legend and amounts.
4. **Average by country** (bar chart, sorted high to low) and **Average by department** (bar chart). Tooltip shows headcount, min, average, max. Clicking a bar sets the matching filter on the Employees page (opens `/employees?country_code=...`).
5. **Salary distribution:** stacked bar chart, one bar per department, segments are salary bands; a switch to view by country. Tooltip shows count and percentage.
6. **Min / average / max by job title:** sortable table (job title, headcount, min, average, max); client-side sort and pagination (about 40 rows).
7. **Outliers:** table of employees paid above the threshold for their title and country (name links to the employee, amount, peer average, ratio), server-side pagination, with a small note explaining the rule (1.5x the peer average, at least 5 peers).

Acceptance:
- Changing the measure or a filter updates all widgets without a full page reload.
- With no data (for example a filter with no employees) widgets show the empty state, not errors or `NaN`.
- All amounts come from `formatUsd`; no floating-point money maths in components.

### 6.4 EmployeesPage (F2)
- **Filter bar:** search box, Country, Department, Job title, Manager (`EmployeeAutocomplete`, searches `/api/employees?q=&page_size=10`), Status (Active default, Inactive, All), and a Clear filters button. "Add employee" button at the top right.
- **Table:** MUI X DataGrid with `paginationMode="server"` and `sortingMode="server"`. Columns: Name, Email, Department, Job title, Country, Manager, Hire date, Base salary, Total compensation, Status. Page sizes 25, 50, 100. Row click opens the detail page. Inactive rows are visually muted with a chip.
- States: skeleton while loading, "No employees match your filters" with a Clear filters action, error state with Retry.
- Acceptance: only one request per change (no duplicate fetches); sort and filter changes reset to page 1; URL reflects the state; the grid never loads more than `page_size` rows.

### 6.5 EmployeeDetailPage (F4, F5, F6)
- **Header:** name, job title, status chip; actions: Edit, Deactivate or Reactivate.
- **Overview card:** department, country, hire date, email, manager (link), direct reports count.
- **Compensation card:** base, bonus, allowances, total in USD; below them the local-currency equivalent (`local`) in a muted style with the text "Approximate, using static rate (updated <date>)".
- **Reporting section:** manager link and a paginated table of direct reports (name link, title, status). When the employee has direct reports, a muted summary line above the table shows the team pay from `GET /api/analytics/team/{id}`: headcount, average and range (for example "Team: 4 people, average $78,400, range $52,000 to $120,000").
- **Salary history:** vertical timeline (a styled list with a line and dots, no extra package), newest first. Each entry shows date, change type chip (Hire, Manual edit, Bulk update), reason, and old to new values for each component that changed (with the percentage change on base). Bulk entries show a short batch id.
- **Deactivate:** confirmation dialog. On 409 (active reports) show the server message in the dialog and keep it open. **Reactivate** similarly handles 409.
- Acceptance: 404 shows a "not found" state with a link back to the list; after any mutation the affected data refreshes.

### 6.6 EmployeeFormPage (F3)
- One form component for create and edit. Fields: full name, email, department, job title, country, manager (autocomplete, optional), hire date (native date input, `TextField type="date"`), base salary, bonus, allowances (dollar inputs).
- Zod schema mirrors backend rules (lengths, email format, hire date not in the future, amounts non-negative with at most 2 decimals, base greater than 0). Convert dollar strings to cents only on submit via `parseDollarsToCents`.
- Beside the salary fields show the local-currency equivalent of the total, computed from the lookups rate, labelled approximate.
- **Edit mode:** loads the employee, sends only changed fields. If a pay field changed, show a "Reason for change" field (optional). Unchanged form disables Save.
- **Server errors:** 409 on email sets a field error on Email; 422 details map to fields; other errors show an alert at the top.
- Acceptance: double submit is prevented; on success navigate to the detail page; leaving with unsaved changes prompts a confirmation.

### 6.7 BulkUpdatePage (F7)
Three-step flow (MUI Stepper):
1. **Target:**
   - Filters: country, department, job title, manager.
   - **What to change** (radio group): Base salary, Bonus, Allowances.
   - **How** (radio group): Percentage change, or Add fixed amount.
   - **Value:** for a percentage, a number input with a `%` suffix (positive or negative, 2 decimals, helper text "Between -50% and +100%"); for a fixed amount, a dollar input with helper text "Annual amount in USD. For $100 per month, enter $1,200."
   - Reason (required).
   - If no filter is set, the user must tick "Apply to all active employees"; this shows a warning.
   - An information note: "Changes apply immediately and update the current annual amount. One-off or monthly payments are not supported."
   - "Preview" button.
2. **Preview:** shows affected employee count, the total of the chosen component before and after (labelled with its name, for example "Total allowances before"), the difference, and a sample table (10 rows) with before and after values. Zero affected shows "No employees would change" and disables the next step. Back keeps the form values.
3. **Confirm:** a dialog states exactly what will happen, for example "Apply +5.00% to the base salary of 123 employees" or "Add $1,200 per year to the allowances of 123 employees". Requires clicking Apply. Sends `expected_count` from the preview.
- On success: show a result summary (count, total delta) with links to the Employees list filtered the same way and to the History page filtered by the new `batch_id` (`/history?batch_id=...`). On `409 stale_preview`: show "The data changed since the preview" and return to step 2 with a fresh preview. On `422 result_too_large`: show the server message on step 1.
- Acceptance: nothing is changed before the final confirm; the Apply button is disabled while the request runs; no way to apply without a preview in the same session; the value input switches between percent and dollar mode without losing the other fields; dollar amounts are converted with `parseDollarsToCents`.

### 6.8 NotFoundPage and error boundary
A top-level error boundary shows a friendly message with a reload button and logs the error to the console only.

### 6.9 HistoryPage (F10)
Purpose: let the HR Manager see every pay change across the organisation and search it.
- **Header:** title "Salary history", subtitle "Every pay change across the organisation".
- **Filter bar:** "Search employee" (name or email), "Search reason", Change type (All, Hire, Manual edit, Bulk update), From and To date fields (native date inputs), Clear filters. When `batch_id` is present, show a removable chip "Batch 6f1c8a" above the table.
- **Table** (MUI X DataGrid, server-side pagination and sorting): Date, Employee (link to the detail page), Type (chip), Reason, Base salary, Bonus, Allowances. A changed component shows `before to after` (for example `$42,857 to $45,000`, with an arrow in the UI) and, in muted text, the percentage change; an unchanged component shows a dash. A hire row shows the initial values only. Sortable by Date (default, newest first), Employee and Type. Page sizes 25, 50, 100. Row click opens the employee.
- **URL state** (`useUrlState`): `employee`, `reason`, `change_type`, `date_from`, `date_to`, `batch_id`, `sort_by`, `sort_dir`, `page`, `page_size`. Text inputs are debounced 300 ms; any filter change resets `page` to 1.
- States: loading skeleton, "No changes match your filters" with Clear filters, error with Retry.
- The acting user is not shown, because it is not recorded.
- Acceptance: one request per change; the URL reflects the state; the grid never loads more than `page_size` rows; opening `/history?batch_id=...` shows exactly the rows of that bulk update.

## 7. Theme and UI conventions
- **Visual source of truth:** `design/DESIGN.md` and the Stitch project (screens named `01 Login` to `07 Not found and errors`; see `design/STITCH_HANDOFF.md`). Colours, typography, spacing and radius go into `theme.js` once.
- **Precedence:** this spec wins for behaviour, data, fields, validation and API use. The Stitch design wins for layout, spacing and styling. If they conflict, follow the spec, build the closest design, and record the difference in `design/STITCH_HANDOFF.md` section 7.
- One MUI theme (`theme.js`): neutral palette, one primary colour, consistent spacing (8 px grid), light mode only.
- Numbers right-aligned in tables; use tabular numerals.
- Use MUI components as they are; custom components only for the shared pieces listed in section 2.

## 8. Frontend tests (Vitest + RTL + MSW)
Test files are named `*.test.jsx` (or `*.test.js`) and sit next to the code they test. Tests are written first (TDD, see `05-seed-and-testing.md` sec 1a): MSW handlers from the API contract, then the page test, then the page.
No real network; MSW handlers mirror the API contract.

| Area | Tests |
|---|---|
| `lib/money.js` | formatting, `parseDollarsToCents` valid and invalid inputs, round trip |
| Auth | login success redirects; 401 shows generic error; 429 message; protected route redirects when signed out; 401 on any call returns to login; logout clears state |
| Employees list | initial load renders rows; typing in search is debounced and sends one request; filter change resets page to 1; URL params restore state; empty and error states |
| Employee form | validation messages; dollars converted to cents in the request; edit sends only changed fields; email conflict shows field error; unsaved-changes prompt |
| Employee detail | history renders old and new values; deactivate 409 message shown; local currency note present; team summary line shown only when the employee has direct reports |
| Bulk update | no apply before preview; "apply to all" required when no filter; choosing component and method shows the matching value input (percent or dollars) with the annual-amount helper text; dollars converted to cents; preview numbers use the chosen component's name; zero affected disables continue; confirm sends `expected_count`; `stale_preview` returns to preview |
| History | filters sent as query params; text inputs debounced; clear filters; batch chip removable; before and after values rendered, dash for unchanged components; hire row shows initial values only; URL params restore state; empty and error states; employee link opens the detail page |
| Dashboard | KPI values formatted; measure toggle refetches; empty data shows empty states |

Target: tests are deterministic (no timers except fake timers for debounce) and the suite runs in under 30 seconds.

## 9. Performance and quality
- Route-level code splitting (`React.lazy`) so the dashboard charts load only when needed.
- Memoise table column definitions; avoid re-renders on each keystroke by debouncing the search.
- Plain JavaScript with ESLint and Prettier in `make lint`. Without compile-time types, rely on tests written first, Zod validation on forms, JSDoc for shared shapes, and the contract in `docs/openapi.json`.
