# PRD: ACME Salary Management

**Author:** Dwipal | **Status:** v2 | **Timebox:** 3 days

## 1. Goal
Replace the Excel sheets the HR team uses today with a web app where the HR Manager can maintain compensation data for ~10,000 employees across several countries and quickly answer "how do we pay people?"

**Persona:** HR Manager, who signs in to the app. More HR users will be added in the future.

**Success looks like:**

- Find or edit any employee.
- Answer common pay questions (by country, department, job title, team) without opening Excel.
- Apply a team-wide raise, bonus or allowance change in one action, with a preview first.
- See how any employee's pay has changed over time.

## 2. Scope and Features

| # | Feature | Notes |
|---|---------|-------|
| F1 | Login / logout | Email + password, server-side session (random session ID in an HttpOnly cookie). One seeded HR user. Users live in an hr_users table so adding more HR users later needs no schema change |
| F2 | Employee list | Server-side pagination, sort, search by name/email, filter by country, department, job title, manager, status |
| F3 | Create / edit employee | Name, email, job title, department, country, hire date, manager, base salary, bonus, allowances (all stored in USD) |
| F4 | Reporting lines | Each employee has at most one manager. The employee page shows the manager and direct reports. The list can be filtered by manager. Validation rejects self-reporting and circular chains |
| F5 | Deactivate employee | Soft delete (status flag), hidden from analytics by default, can be reactivated. An employee with active direct reports cannot be deactivated until those reports are reassigned |
| F6 | Salary history | Every compensation change (hire, manual edit, bulk raise) writes a history row with effective date, old/new values and a reason. Shown as a timeline on the employee page |
| F7 | Bulk pay update | Apply a percentage change or add a fixed annual amount to base salary, bonus or allowances for a filtered group (e.g. +5% base for Engineering in India, or +$1,200 allowances for Sales). Shows affected count and a preview before confirming. Takes effect immediately. Runs in one transaction and writes history rows |
| F8 | Pay insights dashboard | Headcount, min / average / max total compensation (USD) by country, department and job title. Salary band distribution. Team view: headcount and average total compensation of a manager's direct reports. Outlier list (e.g. >1.5x the average for the same title and country) |
| F9 | Seed script | 10,000 realistic employees with a manager hierarchy and salary history across 6 countries: India, US, UK, Germany, Canada, Singapore. Deterministic (fixed random seed) |

## 3. Assumptions and Decisions

- **Compensation = base salary + bonus + allowances**, all annual amounts. Total compensation is the sum of the three. Equity and deductions are not modelled.
- **Pay structure:** the three components are fixed and stored as columns on the employee row, not as a configurable list of components.
- **Currency:** all amounts are stored in USD. Where required (e.g. on an employee's page), they are converted to the employee's local currency for display using a static rate table kept in the DB (easy to update). Conversion is display-only and never changes the stored value. No live Foreign Exchange API.
- **Auth:** single HR user for now, passwords hashed, token-based sessions. No roles, sign-up or password reset yet.
- **Org structure:** each employee has one department and at most one manager (a single reporting line). The top of the hierarchy has no manager. Dotted-line or matrix reporting is not modelled.

## 4. Out of scope

| Out of scope | Reason |
|--------------|--------|
| Payroll, payslips, tax, payments | A different problem domain |
| Salary deductions for excess leaves | Needs leave and attendance data and country-specific leave rules, which is a separate module |
| Audit trail (who changed what) | Salary history records what changed and when, but not the acting user. Actor tracking can be added later now that hr_users exists |
| Excel/CSV export | For now out of scope. However we can implment if required |
| Multi-user management, roles, permissions | Only one HR user is needed today. The data model allows it later |
| Median and percentile statistics | Average, min, max and bands answer the core pay questions. See trade-offs |
| Org chart visualisation and full-subtree rollups | Direct reports are enough to answer team-level pay questions |
| Live Foreign Exchange rates, country tax or statutory rules | Static rates are enough for showing local-currency equivalents |
| Configurable pay structure (custom pay components such as stock, housing or relocation, or components that differ by country) | The PRD fixes the components as base, bonus and allowances, stored on the employee row. Configurable components need a separate components table and are not required to answer the core pay questions. See trade-offs |

## 5. Technical Approach

- **Backend:** FastAPI + SQLAlchemy + SQLite (chosen for simplicity, as this is an interview task), Pydantic validation, layered structure (routes / services / repositories). Server-side sessions kept in a sessions table (HttpOnly, Secure, SameSite cookie), enforced by a dependency that protects all employee routes.
- **Frontend:** React (Vite) + a component library (MUI or shadcn/ui), with a login page, protected routes and a table with server-side pagination.
- **Data model:** hr_users, employees (with a self-referencing manager_id; base salary, bonus and allowances are columns on the same row), salary_history, currency_rates.
- **Performance:** indexes on country, department, job title, status, email, manager_id and salary_history.employee_id. Aggregations run in SQL. Page size is capped (e.g. 100).
- **Tests:** fast, deterministic unit tests on services (bulk update math, history creation, manager validation, currency conversion, stats, auth) plus a few API tests against an in-memory SQLite DB.

## 6. Risks and Trade-offs

- **SQLite (chosen as this is an interview task).** It has no native median or percentiles, so the dashboard uses min, average, max and bands. It also has limited concurrent writes, and a production version would move to Postgres.
- **USD storage with static FX rates.** Storing everything in USD keeps comparisons and analytics simple, but local-currency amounts are approximate, do not reflect rate changes over time, and may differ from what an employee is actually paid locally. Accepted to keep the model simple and deterministic.
- **No actor in salary history.** We can show what changed and when, but not who changed it. Accepted for this iteration, see section 4.
- **Single HR user, no roles.** Less auth complexity, but no access separation today. A roles layer is needed when more HR users are added.
- **Direct reports only for team views.** Simple queries and easy tests, but no pay rollup for a manager's whole organisation (all levels below them).
- **Fixed pay components on the employee row.** Base, bonus and allowances as columns keep analytics a simple sum with no joins, and let the database enforce types and rules. The cost is that adding a new type of pay later needs a schema change, and HR cannot define its own components. If components become many or differ by country, they would move to a separate table with one row per employee and component.
- **Bulk update safety.** A change could be applied to the wrong group. Mitigation: mandatory preview with affected count, transactional apply, and history rows that make every change traceable and reversible by hand.
- **Performance at 10,000 employees.** Mitigation: indexes, server-side pagination with a page-size cap, SQL aggregations, and checking timings on the seeded data.
- **Invalid reporting lines** (self-reporting, circular chains, orphaned reports after a deactivation). Mitigation: validation on create/edit, blocking deactivation of a manager with active reports, and unit tests for each case.
