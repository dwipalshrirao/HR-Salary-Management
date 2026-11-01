# Stitch to OpenCode Handoff

How the AI coding agent (OpenCode) uses the Stitch design while building the React frontend, and how the design lines up with the specs.

## 1. Purpose and precedence
- **Stitch design = visual reference.** Layout, spacing, colour, typography, component look.
- **Specs = behaviour.** Fields, validation, filters, API calls, states, rules (`specs/04-frontend-spec.md`, `specs/02-backend-spec.md`, `specs/03-analytics-spec.md`).
- If they conflict: follow the spec, build the closest visual match, and add a row to section 7 below.
- Never paste Stitch HTML/CSS into the app. Rebuild every screen with MUI, MUI X DataGrid and Recharts, with tokens from `theme.js` (see `stitch-prompts.md`, last sections).

## 2. Screen map
Screen names in Stitch must match exactly (rename after generation).

| Stitch screen | Route | Spec section | Main components | Data (API) |
|---|---|---|---|---|
| (app shell, on every screen) | n/a | 04 sec 6.2 | AppBar, Tabs/links, user menu | `GET /api/auth/me`, `POST /api/auth/logout` |
| `01 Login` | `/login` | 04 sec 6.1 | Card, TextField, Button, Alert | `POST /api/auth/login` |
| `02 Dashboard` | `/` | 04 sec 6.3, 03 | StatCard, Recharts bar and stacked bar, Table, toggle, Select | `/api/analytics/summary`, `breakdown`, `distribution`, `outliers` |
| `03 Employees` | `/employees` | 04 sec 6.4 | Filter bar, DataGrid (server mode), Autocomplete, Chip | `GET /api/employees`, `GET /api/lookups` |
| `04 Employee detail` | `/employees/:id` | 04 sec 6.5 | Cards, Chip, Table, styled-list timeline, Dialog | `GET /api/employees/{id}`, `/history`, `/direct-reports`, `POST .../deactivate`, `.../reactivate` |
| `05 Employee form` | `/employees/new`, `/employees/:id/edit` | 04 sec 6.6 | React Hook Form, TextField, Select, Autocomplete, `type="date"` | `POST /api/employees`, `PATCH /api/employees/{id}` |
| `06 Bulk update` | `/bulk-update` | 04 sec 6.7 | Stepper, Select, RadioGroup, TextField, Checkbox, Alert, Dialog, Table | `POST /api/bulk-updates/preview`, `/apply` |
| `07 Not found and errors` | `*` and error boundary | 04 sec 6.8 | Centered message, Button | n/a |
| `08 History` | `/history` | 04 sec 6.9 | Filter bar, DataGrid (server mode), Chip | `GET /api/salary-history` |

## 3. Connect Stitch to OpenCode (MCP)
1. Complete Stitch's own MCP setup for your account. It gives you the MCP endpoint and the credential to use. I could not confirm these values from public documentation, so take them from that setup page, not from this file.
2. Add the server to the project's `opencode.json` (project config overrides global config):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "stitch": {
      "type": "remote",
      "url": "<STITCH_MCP_URL from the Stitch setup page>",
      "enabled": true
    }
  }
}
```
3. If the setup asks for an API key or a header, add it under `headers` and reference an environment variable (for example `STITCH_API_KEY`) using OpenCode's environment-variable syntax. Do not paste the key into the file. Keep `opencode.json` free of secrets, or gitignore the file if it has to contain any.
4. Restart OpenCode and check the connection with `opencode mcp debug stitch` (and `opencode mcp auth stitch` if the server uses a browser sign-in).
5. Smoke test with these prompts:
   - "List the Stitch projects I can access."
   - "Fetch the screen named `02 Dashboard` from the ACME Salary Management project and describe its layout."
6. Optional: Google publishes agent skills for Stitch (the `google-labs-code/stitch-skills` repository, including ones that turn screens into React components). For this project, use them only to fetch and read designs. Our specs choose MUI, so do not let a skill switch the UI library or add dependencies.

## 4. Workflow for the agent (every frontend page)
1. Read the page's spec section (table in section 2).
2. Fetch the Stitch screen by name. Read all its frames (error, empty, loading and dialog states).
3. Note the layout, spacing and visual details. Check the tokens against `design/DESIGN.md`; if a colour or size differs, prefer `DESIGN.md` and `theme.js`.
4. Build the page from the shared components in the spec (`StatCard`, `Money`, `ConfirmDialog`, `EmptyState`, `ErrorState`) and MUI. Do not add new libraries.
5. Compare the result with the Stitch screen and the checklist in `stitch-prompts.md`.
6. Commit with the screen name in the message, for example `feat(F2): employees page with filters and DataGrid (spec 04 sec 6.4, stitch "03 Employees")`. Write the page tests first, as in `specs/05-seed-and-testing.md` sec 1a.

## 5. What to take from Stitch and what not to
| Take from Stitch | Take from the specs |
|---|---|
| Page layout, card arrangement, spacing | Field lists, validation, error messages |
| Visual hierarchy, typography, colours | Filters, sort options, page sizes, URL parameters |
| How states look (loading, empty, error, dialogs) | API calls, query keys, cache invalidation |
| Chart styling and colours | Chart data, bands, measures |
| Sample copy for labels and buttons | Money format (`lib/money.js`), dates, rules |

Ignore anything in Stitch that implies a feature outside the specs (extra filters, export buttons, charts, roles, notifications).

Common extras Stitch adds, to ignore when building: notification bell, global search in the top bar, settings or help links, extra avatar-menu items, export or download buttons, extra KPI cards or chart types, "trend" arrows or percentage changes that the analytics endpoints do not return, extra table columns, sidebars, footers, and comment or share buttons.

## 6. Alignment checklist (design and specs)
Checked against `specs/04-frontend-spec.md`:

| Item | Design | Spec | Status |
|---|---|---|---|
| Login errors (401, 429) | frames: error, rate-limited | sec 6.1 | Aligned |
| Nav links and user menu with Logout | app shell | sec 6.2 | Aligned |
| Dashboard controls: measure toggle, country, department | `02 Dashboard` | sec 6.3 | Aligned |
| KPI cards, pay mix, averages by country and department | `02 Dashboard` | sec 6.3 | Aligned |
| Distribution chart with By department / By country toggle | `02 Dashboard` | sec 6.3 | Aligned (toggle added) |
| Job-title min/avg/max table and outlier table with rule note | `02 Dashboard` | sec 6.3 | Aligned |
| Employees filters, columns, page sizes, muted inactive rows | `03 Employees` | sec 6.4 | Aligned |
| Detail: profile, compensation, local-currency note, reporting, history | `04 Employee detail` | sec 6.5 | Aligned |
| Deactivate dialog with 409 error and Reactivate | `04 Employee detail` | sec 6.5 | Aligned (inactive frame added) |
| Form fields, local total, reason field, discard-changes dialog | `05 Employee form` | sec 6.6 | Aligned (discard frame added) |
| Bulk flow: component and method choice, percent or fixed amount, preview, confirm, result, stale preview | `06 Bulk update` | sec 6.7 | Aligned |
| Not found and error pages | `07 Not found and errors` | sec 6.8 | Aligned (prompt added) |
| History page: filters (employee, reason, type, dates), before/after values, batch chip | `08 History` | sec 6.9 | Aligned |
| Currency rates | no screen (API only) | 02 sec 5.8 | Intentionally not designed |
| Money formats: `$45,000`, compact `$812.3M`, local `₹4,257,900` | all | sec 5 | Aligned (compact and local formats added to spec) |
| Timeline and date input without extra packages | detail, form | sec 6.5, sec 6.6 | Aligned (styled list and native date input) |
| Click-through from dashboard bars to a filtered employee list | not drawn in Stitch | sec 6.3 | Spec only; implement as described |
| Unsaved-changes prompt, double-submit protection, URL-synced filters | partly shown | sec 5, sec 6.4, sec 6.6 | Spec only |

## 7. Deviations log
Add a row whenever the built UI differs from the Stitch screen or the spec.

| Date | Screen | Difference | Decision and reason |
|---|---|---|---|
| | | | |
