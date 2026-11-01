# Google Stitch Prompts: ACME Salary Management

Use with `DESIGN.md` (same folder). Stitch reads a DESIGN.md as persistent design rules, so add it to the project's design system settings if your Stitch version offers that; otherwise paste its full contents at the top of your first prompt.

## How to use
1. Create a new **Web** project in Stitch and load `DESIGN.md` as above.
2. Run **Prompt 1** (login), then the others one at a time. Generating a screen per prompt gives better results than asking for everything at once.
3. Check each result against the checklist at the end. If Stitch added elements that are not in the prompt, remove them with a short follow-up such as "Remove the notification icon and the export button. Do not add anything else." Edit the existing screen instead of regenerating it. Fix with short follow-up prompts ("make the table header lighter", "reduce card padding").
4. Treat Stitch output as the **visual reference**. Build the real screens in React with MUI (mapping at the end), not by pasting Stitch's HTML.

Every prompt follows the same shape: purpose and mood, design system reminder, page structure, sample data, states.

**Name every screen in Stitch exactly like this** (rename them after generation). OpenCode will look screens up by these names through the MCP connection, and `STITCH_HANDOFF.md` uses the same names.

| Stitch screen name | Frames (suffix after the name) |
|---|---|
| `01 Login` | default, error, loading, rate-limited |
| `02 Dashboard` | default, loading, empty |
| `03 Employees` | default, loading, empty, error |
| `04 Employee detail` | default, deactivate dialog, deactivate error, inactive employee |
| `05 Employee form` | create, validation errors, email conflict, edit, discard dialog |
| `06 Bulk update` | target (percentage), target (fixed amount), preview, confirm, result, stale preview |
| `07 Not found and errors` | not found, something went wrong |
| `08 History` | default, loading, empty, error, filtered by batch |

---

## Prompt 1: Login

```
Design a desktop web LOGIN screen for "ACME Salary Management", an internal HR tool. Mood: calm, trustworthy, minimal. Follow the attached design system (blue accent, Inter, flat white cards on a light grey background).

Layout:
1. Centered card, 420 px wide, vertically centered on a light grey page. No top navigation.
2. Card content: product name "ACME Salary Management" as a small title, heading "Sign in", one line of Slate helper text ("Use your HR account to continue").
3. Email field and Password field (outlined, with labels). A show/hide password icon.
4. Full-width primary button "Sign in".
5. Below the button, an error alert area showing the text "Invalid email or password".

Show the default state. Add a second frame with the error alert visible, a third with the button in a loading state (spinner, disabled), and a fourth with the alert text "Too many attempts. Try again in a few minutes."

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 2: Dashboard (pay insights)

```
Design a desktop web DASHBOARD called "Pay insights" for an HR Manager. Mood: data-first, calm, professional. Follow the attached design system. Use realistic sample data in USD.

App shell: white top bar with "ACME Salary Management" on the left, links Dashboard (active), Employees, History, Bulk update, and a user menu on the right showing the signed-in name with a Logout option.

Page structure:
1. Page header: title "Pay insights", subtitle "Active employees only". On the right: a toggle "Total compensation | Base salary", and two dropdowns "All countries" and "All departments".
2. Four KPI cards in one row: "Total compensation expenditure" $812.3M; "Headcount" 9,700; "Average compensation" $83,740; "Highest-paying country" United States with caption "$118,200 average".
3. A card "Pay mix" with one horizontal stacked bar showing base 82%, bonus 11%, allowances 7%, with legend and dollar amounts.
4. Two charts side by side: "Average compensation by country" (vertical bars, sorted high to low: United States, Singapore, Canada, Germany, United Kingdom, India) and "Average compensation by department" (horizontal bars: Engineering, Product, Sales, Finance, Marketing, Human Resources, Operations, Customer Support).
5. A full-width card "Salary distribution" with a small toggle in the card header "By department | By country" (department selected): stacked bar per department with 7 ordered salary bands (under $30k up to $200k+) in a light-to-dark blue scale, with a legend.
6. A table card "Pay by job title" with columns: Job title, Headcount, Minimum, Average, Maximum (numbers right-aligned, sortable headers, about 8 visible rows, pagination).
7. A table card "Pay outliers" with a one-line note "Paid more than 1.5x the average for the same job title and country (at least 5 peers)". Columns: Employee, Job title, Country, Compensation, Peer average, Ratio (for example 1.9x). Pagination at the bottom.

Add a second frame showing loading skeletons and a third frame showing the empty state for a filter with no employees.

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 3: Employees list

```
Design a desktop web EMPLOYEES LIST screen for an HR Manager managing 10,000 employees. Mood: efficient, scannable, calm. Follow the attached design system. Same app shell as the dashboard with "Employees" active.

Page structure:
1. Page header: title "Employees", result text "9,700 employees", and a primary button "Add employee" on the right.
2. Filter bar in a white card: search field ("Search name or email"), dropdowns for Country, Department, Job title, a Manager autocomplete field, a Status dropdown (Active selected), and a text button "Clear filters".
3. Data table (MUI DataGrid style) with columns: Name, Email, Department, Job title, Country, Manager, Hire date, Base salary, Total compensation, Status. Numbers right-aligned. Sortable headers with arrows. Row height 44 px, subtle hover. Show 10 realistic rows, such as Asha Patel (Engineering, Senior Software Engineer, India, manager Ravi Shah, $45,000 base, $51,300 total, Active), Michael Chen (Sales, Account Executive, United States), Priya Nair, Lukas Meyer (Germany), Sophie Turner (United Kingdom), Wei Tan (Singapore), Emma Wilson (Canada). Include one Inactive row shown muted with a grey chip.
4. Table footer: "Rows per page: 25" selector, range text "1-25 of 9,700", previous and next buttons.

Add frames for: loading skeleton rows, empty state ("No employees match your filters" with a "Clear filters" button), and an error state with a "Retry" button.

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 4: Employee detail

```
Design a desktop web EMPLOYEE DETAIL screen. Mood: clear, organised, calm. Follow the attached design system. Same app shell, "Employees" active. Sample employee: Asha Patel, Senior Software Engineer, Engineering, India.

Page structure:
1. Breadcrumb "Employees / Asha Patel". Header with name, job title, a green "Active" chip, and on the right an outlined "Edit" button and an outlined red "Deactivate" button.
2. Two cards side by side. Left "Profile": email, department, country, hire date (12 Apr 2021), manager (a blue link "Ravi Shah"), direct reports count (0). Right "Compensation (USD)": Base $45,000, Bonus $4,500, Allowances $1,800, Total $51,300 (bold). Under it, a small Slate line "Approximate: ₹4,257,900 INR (static rate, updated 1 Jan 2026)".
3. Card "Reporting": manager link and, for a manager example, above a small table of direct reports (Name, Job title, Status) with pagination, a muted line "Team: 4 people, average $78,400, range $52,000 to $120,000" (shown only for managers). For this employee show the empty text "No direct reports".
4. Card "Salary history": vertical timeline, newest first. Entries: "31 Oct 2026, Bulk update, FY27 raise, base $45,000 from $42,857 (+5.0%)"; "1 Mar 2025, Manual edit, Annual review, base $42,857 from $40,000 (+7.1%)"; "12 Apr 2021, Hire, base $40,000". Change-type chips: Hire (blue), Manual edit (grey), Bulk update (teal).

Add a second frame: a "Deactivate employee?" confirmation dialog, a third frame where the dialog shows an error: "This employee has 4 active direct reports. Reassign them first.", and a fourth frame showing an INACTIVE employee: muted grey "Inactive" chip and a "Reactivate" button in place of Deactivate.

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 5: Employee form (create and edit)

```
Design a desktop web CREATE EMPLOYEE form screen (the same layout is reused for editing). Mood: simple, clear, low-effort. Follow the attached design system. Same app shell, "Employees" active.

Page structure:
1. Breadcrumb "Employees / New employee", page title "Add employee".
2. One white card with a two-column form, grouped by headings:
   - "Personal and job": Full name, Email, Department (select), Job title (select), Country (select), Manager (autocomplete, optional), Hire date (standard date input).
   - "Compensation (annual, USD)": Base salary, Bonus, Allowances (dollar inputs with a $ prefix). Beside them a Slate helper line: "Total: $51,300 (approximate: ₹4,257,900 INR)".
3. Footer actions inside the card: outlined "Cancel" and primary "Save employee".

Frames: default empty; validation errors (red helper text under Email "Enter a valid email" and Base salary "Enter an amount greater than 0"); a conflict error on Email ("This email is already in use"); and an EDIT version titled "Edit Asha Patel" with values filled in and an extra optional field "Reason for change" that appears when a compensation field is modified. Add one more frame: a "Discard changes?" dialog with buttons "Keep editing" and "Discard".

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 6: Bulk pay update (3 steps)

```
Design a desktop web BULK PAY UPDATE flow with three steps using a horizontal stepper (Target, Preview, Confirm). Mood: careful, safe, clear. This is a high-impact action, so the design should feel deliberate. Follow the attached design system. Same app shell, "Bulk update" active.

Frame 1, "Target": a white card with filter dropdowns (Country, Department, Job title, Manager); a radio group "What to change" with options Base salary, Bonus, Allowances; a radio group "How" with options "Percentage change" and "Add fixed amount"; a value field (Percentage change selected: a number field with % suffix, helper text "Between -50% and +100%"); a "Reason" text field; and a checkbox "Apply to all active employees" with an amber warning alert beneath it when no filter is set. A small info note: "Changes apply immediately and update the current annual amount. One-off or monthly payments are not supported." Primary button "Preview".

Frame 2, same step with "Add fixed amount" selected and Allowances chosen: the value field is a dollar field with $ prefix and helper text "Annual amount in USD. For $100 per month, enter $1,200."

Frame 3, "Preview" (base salary, +5%): summary cards "Employees affected 123", "Total base salary before $5.54M", "Total base salary after $5.82M", "Difference +$277,000". Below, a table of 10 sample employees with columns Name, Base before, Base after. Buttons "Back" and "Continue".

Frame 4, "Confirm": a dialog titled "Apply this change?" with text "Apply +5.00% to the base salary of 123 employees in Engineering, India." Buttons "Cancel" and a primary "Apply to 123 employees".

Frame 5, result: success alert "Updated 123 employees" with links "View in employee list" and "View changes in history". Frame 6: warning alert "The data changed since the preview. Please preview again."

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 7: Not found and errors

```
Design two simple desktop web screens for "ACME Salary Management". Mood: calm, helpful. Follow the attached design system. Same app shell.

Frame 1, "Not found": centered content with a simple line icon, heading "Page not found", one Slate sentence ("The page or employee you are looking for does not exist.") and a primary button "Back to employees".

Frame 2, "Something went wrong": centered content with a simple line icon, heading "Something went wrong", one Slate sentence ("Reload the page and try again.") and a primary button "Reload".

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

## Prompt 8: History

```
Design a desktop web HISTORY screen listing every salary change across the organisation. Mood: clear, scannable, calm. Follow the attached design system. Same app shell with "History" active.

Page structure:
1. Page header: title "Salary history", subtitle "Every pay change across the organisation".
2. Filter bar in a white card: a search field "Search employee (name or email)", a search field "Search reason", a dropdown "Change type" (All, Hire, Manual edit, Bulk update), two date fields "From" and "To", and a text button "Clear filters".
3. Data table (DataGrid style) with columns: Date, Employee (blue link), Type (chip), Reason, Base salary, Bonus, Allowances. For a changed component show "$42,857 → $45,000" with a small Slate "+5.0%" below it; for an unchanged component show "–". Show about 8 rows, for example: 31 Oct 2026, Asha Patel, Bulk update, "FY27 raise", base +5.0%; 31 Oct 2026, Michael Chen, Bulk update, "FY27 raise", base +5.0%; 15 Oct 2026, Priya Nair, Manual edit, "Promotion", base and bonus changed; 1 Mar 2025, Lukas Meyer, Manual edit, "Annual review"; 12 Apr 2021, Sophie Turner, Hire, "New hire", initial values only with no arrows. Sortable headers on Date, Employee and Type.
4. Table footer: "Rows per page: 25" selector, range text "1-25 of 28,430", previous and next buttons.

Add frames for: loading skeleton rows; empty state ("No changes match your filters" with a "Clear filters" button); error state with a "Retry" button; and a frame filtered by one bulk update, showing a removable chip "Batch 6f1c8a" above the table.

SCOPE: include only the elements listed above, with exactly the counts, labels and navigation items given. Do not add anything else: no extra navigation items, notifications, global search, settings, help, avatar-menu items other than Logout, export or import buttons, extra filters, extra KPI cards, extra charts, tabs, sidebars, banners, footers or promotional content. If an element is not listed, leave it out.
```

---

## After generating: map the design to React

Build the app with the component library chosen in the specs. Use Stitch only as the visual guide.

| Design element | React implementation |
|---|---|
| App shell, top bar | MUI `AppBar` + `Toolbar`, `Tabs` or `Button` links |
| Page layout and grid | MUI `Container` (maxWidth 1280) + `Grid` |
| Cards | MUI `Card` with `variant="outlined"` |
| KPI cards | Custom `StatCard` built on `Card` |
| Buttons, fields, selects | MUI `Button`, `TextField` (size small), `Select`, `Autocomplete` |
| Employee table | MUI X `DataGrid` (server-side pagination and sorting) |
| Small tables (job title, outliers, direct reports) | MUI `Table` or `DataGrid` with client pagination |
| Status and change-type chips | MUI `Chip` (tonal colours from the design system) |
| Charts | Recharts `BarChart`, stacked `BarChart` |
| Stepper | MUI `Stepper` |
| Component and method choices (bulk update) | MUI `RadioGroup` with `FormControlLabel` |
| Dialogs, alerts, snackbar | MUI `Dialog`, `Alert`, `Snackbar` |
| Timeline (salary history) | Simple styled list with a vertical line and dots (no extra package) |
| Date input | MUI `TextField` with `type="date"` |
| Loading | MUI `Skeleton` |

Set the design tokens once in `theme.js`:
```js
import { createTheme } from "@mui/material/styles";

export const theme = createTheme({
  palette: {
    primary: { main: "#1E4ED8", dark: "#173DAE", light: "#E8EEFC" },
    secondary: { main: "#0F766E" },
    success: { main: "#2E7D32" },
    warning: { main: "#B45309" },
    error: { main: "#C62828" },
    background: { default: "#F6F7F9", paper: "#FFFFFF" },
    text: { primary: "#1A2230", secondary: "#5B6676", disabled: "#9AA3B2" },
    divider: "#E3E7EE",
  },
  shape: { borderRadius: 8 },
  spacing: 8,
  typography: {
    fontFamily: "Inter, system-ui, sans-serif",
    h1: { fontSize: 28, lineHeight: "36px", fontWeight: 600 },
    h2: { fontSize: 20, lineHeight: "28px", fontWeight: 600 },
    h3: { fontSize: 16, lineHeight: "24px", fontWeight: 600 },
    body1: { fontSize: 14, lineHeight: "20px" },
    caption: { fontSize: 12, lineHeight: "16px" },
  },
  components: {
    MuiButton: { styleOverrides: { root: { textTransform: "none", borderRadius: 6 } } },
    MuiCard: { defaultProps: { variant: "outlined" } },
    MuiTextField: { defaultProps: { size: "small" } },
  },
});
```
Use `font-variant-numeric: tabular-nums` on table cells and KPI values.

## Review checklist for each generated screen
- Uses the blue accent only; no extra accent colours outside charts.
- One primary button per view; destructive actions are red.
- Numbers right-aligned, with a thousands separator, and amounts labelled as USD.
- Shows loading, empty and error states.
- Every element can be built with the components in the table above (no custom widgets).
- Nothing extra: every element on the screen appears in the prompt. Anything else (notifications, global search, export buttons, extra cards or filters) is removed with a follow-up prompt.
- Matches the specs: list filters, columns and bulk-update steps from `specs/04-frontend-spec.md`.
