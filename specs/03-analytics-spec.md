# 03. Analytics Specification (F8)

Goal: let the HR Manager answer "how do we pay people?" in a few clicks. This spec turns the interviewer's example questions into concrete metrics and endpoints.

## 1. Question-to-metric map

| HR question | Metric | Endpoint |
|---|---|---|
| What is the total salary expenditure of the organisation? | Sum of compensation, with base / bonus / allowances split | `GET /api/analytics/summary` |
| What is the average salary by country or department? | Headcount, average per group | `GET /api/analytics/breakdown?group_by=country\|department` |
| What are the minimum and maximum salary by job title? | Min, average, max per group | `GET /api/analytics/breakdown?group_by=job_title` |
| Which countries or departments have the highest average salaries? | Same breakdown, sorted by average, descending | `breakdown` with `sort_by=avg&sort_dir=desc&limit=N` |
| How are salaries distributed across departments? | Headcount per salary band, per department or country | `GET /api/analytics/distribution` |
| How much does a manager's team cost and what is the range? | Direct reports headcount, sum, average, min, max | `GET /api/analytics/team/{manager_id}` |
| Who is paid unusually high for their role and country? | Outlier list | `GET /api/analytics/outliers` |

## 2. Common definitions
- **Population:** active employees only. `include_inactive=true` adds inactive ones.
- **Measure:** `measure=total` (default, base + bonus + allowances, column `total_comp_cents`) or `measure=base` (column `base_salary_cents`). The measure is mapped to a column from a whitelist in code.
- **Currency:** USD, in integer cents. Averages are rounded to the nearest cent (half up) in the service.
- **Optional scope filters** accepted by every endpoint unless stated: `country_code`, `department_id`, `job_title_id`.
- **Empty results:** a group or dataset with no employees returns zeros and empty lists, never an error.
- **No median or percentiles** (out of scope, see PRD).

## 3. Endpoints

### 3.1 Summary `GET /api/analytics/summary`
Params: `measure`, `include_inactive`, scope filters.
```json
{
  "headcount": 9700,
  "total_cents": 0, "avg_cents": 0, "min_cents": 0, "max_cents": 0,
  "components": { "base_cents": 0, "bonus_cents": 0, "allowances_cents": 0 },
  "country_count": 6, "department_count": 9
}
```
`total_cents` is the sum of the chosen measure. `components` always shows the three pay components for the same population, so the UI can show the pay mix.

SQL sketch:
```sql
SELECT COUNT(*), SUM(m), AVG(m), MIN(m), MAX(m),
       SUM(base_salary_cents), SUM(bonus_cents), SUM(allowances_cents),
       COUNT(DISTINCT country_code), COUNT(DISTINCT department_id)
FROM employees WHERE status = 'active' /* + scope filters */;
```
(`m` is the whitelisted measure column.)

### 3.2 Breakdown `GET /api/analytics/breakdown`
Params: `group_by` (`country` | `department` | `job_title`, required), `measure`, `include_inactive`, scope filters, `sort_by` (`name` | `headcount` | `total` | `avg` | `min` | `max`, default `avg`), `sort_dir` (default `desc`), `limit` (1 to 100, optional).
```json
{ "group_by": "country", "measure": "total",
  "rows": [ { "key": "US", "name": "United States", "headcount": 2700,
              "total_cents": 0, "avg_cents": 0, "min_cents": 0, "max_cents": 0 } ] }
```
`key` is the country code or the numeric id (as a string). Sorting ties break by `name`.

SQL sketch (department):
```sql
SELECT d.id, d.name, COUNT(*) AS headcount,
       SUM(e.m) AS total, AVG(e.m) AS avg, MIN(e.m) AS min, MAX(e.m) AS max
FROM employees e JOIN departments d ON d.id = e.department_id
WHERE e.status = 'active' /* + scope filters */
GROUP BY d.id, d.name
ORDER BY <whitelisted sort expression> <dir>, d.name
LIMIT :limit;
```
Country and job title are the same query with their own table. Only the join table and group columns differ; build it from a small mapping, not three copies.

### 3.3 Distribution `GET /api/analytics/distribution`
Params: `group_by` (`none` | `department` | `country`, default `none`), `measure`, `include_inactive`, scope filters.

Bands (annual USD, edges in cents, defined once in `analytics/bands.py`): `< 30k`, `30k to < 50k`, `50k to < 75k`, `75k to < 100k`, `100k to < 150k`, `150k to < 200k`, `200k and above`. A value exactly on an edge belongs to the higher band.
```json
{ "measure": "total",
  "bands": [ { "index": 0, "label": "< $30k" }, { "index": 1, "label": "$30k - $50k" } ],
  "groups": [ { "key": "all", "name": "All employees", "headcount": 9700,
                "counts": [120, 800, 2100, 2500, 2400, 1200, 580] } ] }
```
`counts` always has one entry per band, zeros included, in band order.

SQL sketch:
```sql
SELECT <group key>, 
       CASE WHEN m < 3000000 THEN 0 WHEN m < 5000000 THEN 1 WHEN m < 7500000 THEN 2
            WHEN m < 10000000 THEN 3 WHEN m < 15000000 THEN 4 WHEN m < 20000000 THEN 5
            ELSE 6 END AS band,
       COUNT(*) AS n
FROM employees e /* join group table */ WHERE e.status = 'active'
GROUP BY <group key>, band;
```
Generate the `CASE` from the band edge list so the SQL and the labels cannot drift. The service pivots rows into the `counts` arrays.

### 3.4 Team `GET /api/analytics/team/{manager_id}`
Direct reports only (active unless `include_inactive=true`). Returns the manager summary and `headcount`, `total_cents`, `avg_cents`, `min_cents`, `max_cents` for the chosen `measure`. 404 if the manager does not exist. A manager with no reports returns zeros.

### 3.5 Outliers `GET /api/analytics/outliers`
Params: `threshold` (decimal, default 1.5, range 1.1 to 5), `min_peers` (int, default 5, range 2 to 100), `measure`, scope filters, `page`, `page_size`.

Rule: an active employee is an outlier when their measure is greater than `threshold` times the average of **active employees with the same job title and country**, and that group has at least `min_peers` people. The employee is included in the group average (documented; keeps the query simple).
```sql
WITH peer AS (
  SELECT job_title_id, country_code, AVG(m) AS avg_m, COUNT(*) AS n
  FROM employees WHERE status = 'active'
  GROUP BY job_title_id, country_code HAVING COUNT(*) >= :min_peers
)
SELECT e.id, e.full_name, jt.name AS job_title, c.name AS country,
       e.m AS amount_cents, CAST(ROUND(p.avg_m) AS INTEGER) AS peer_avg_cents,
       e.m * 1.0 / p.avg_m AS ratio
FROM employees e
JOIN peer p ON p.job_title_id = e.job_title_id AND p.country_code = e.country_code
JOIN job_titles jt ON jt.id = e.job_title_id
JOIN countries c ON c.code = e.country_code
WHERE e.status = 'active' AND e.m > :threshold * p.avg_m
ORDER BY ratio DESC, e.id
LIMIT :page_size OFFSET :offset;
```
Response: standard pagination envelope; item = `{ employee_id, full_name, job_title, country, amount_cents, peer_avg_cents, ratio }` with `ratio` rounded to 2 decimals.

## 4. Implementation notes
- One `AnalyticsService` with small private helpers: `_measure_column(measure)`, `_scope_filters(params)`, `_population(include_inactive)`. Endpoint-specific methods compose them.
- Reuse the same filter builder as the employee list where the filters overlap.
- Use SQLAlchemy Core expressions (`func.sum`, `case`) so all values are bound parameters. Never format user text into SQL.
- All endpoints are read-only and safe to cache in the browser for 60 seconds.
- Expected cost on 10,000 rows is a few milliseconds per query (single scan or index range); target under 100 ms per request.

## 5. Tests (golden dataset)
Build a small fixed dataset in `tests/factories.py` (about 14 employees across 3 countries, 3 departments, 4 titles, 2 inactive) and hand-compute the expected numbers in the test, not by calling the code under test.

| Test | Asserts |
|---|---|
| summary totals | headcount, sum, avg, min, max, components match hand calculation |
| inactive excluded by default, included with flag | counts change as expected |
| measure=base vs total | different sums, same headcount |
| breakdown by each dimension | rows, headcount, min/avg/max per group; sorting by avg desc; `limit` |
| ties and empty groups | tie broken by name; empty scope returns empty rows |
| distribution edge values | value exactly on an edge lands in the higher band; counts length is always 7 |
| distribution per department | per-group counts sum to group headcount |
| team stats | only direct reports counted; manager with no reports returns zeros; unknown manager 404 |
| outliers | threshold and `min_peers` respected; sorted by ratio; pagination; inactive ignored |
| scope filters | `country_code` and `department_id` narrow every endpoint consistently |
| rounding | averages round half up to whole cents |
| validation | bad `group_by`, `threshold` out of range, `limit` too large return 422 |

Performance smoke (marked `slow`, runs on the 10k seed): each analytics endpoint answers in under 100 ms.
