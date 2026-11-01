# Design System: ACME Salary Management

**Product:** Internal web app for an HR Manager to maintain compensation data for about 10,000 employees and understand how the organisation pays people.
**Platform:** Desktop web first (1440 px wide design, usable from 1024 px). Light mode only.
**Implementation target:** React with MUI (Material UI) and MUI X DataGrid, Recharts for charts. Design with components that exist in MUI so the result can be built without custom widgets.

## 1. Visual Theme and Atmosphere
Calm, trustworthy and efficient, like a modern finance or HR back-office tool. Dense enough to scan thousands of rows, but never cramped. Lots of white space around data, quiet colour, and one confident blue accent. Numbers are the hero: large, aligned and easy to compare. No decoration for its own sake: no gradients, no illustrations except simple empty states, no heavy shadows.

Keywords: professional, precise, calm, data-first, neutral, approachable.

## 2. Colour Palette and Roles

| Role | Name | Hex | Use |
|---|---|---|---|
| Primary | Ledger Blue | `#1E4ED8` | Primary buttons, links, active nav item, selected states, focus ring |
| Primary dark | Deep Blue | `#173DAE` | Button hover, pressed states |
| Primary tint | Mist Blue | `#E8EEFC` | Selected row, hovered nav, info backgrounds |
| Secondary | Teal | `#0F766E` | Secondary data series, positive highlights |
| Success | Green | `#2E7D32` | Active status, successful results |
| Warning | Amber | `#B45309` | Cautions (organisation-wide change, stale data) |
| Error | Red | `#C62828` | Errors, destructive actions (Deactivate) |
| Background | Cloud | `#F6F7F9` | Page background |
| Surface | White | `#FFFFFF` | Cards, tables, dialogs, app bar |
| Table header | Fog | `#F1F4F9` | Data table header row |
| Border | Line | `#E3E7EE` | Card borders, dividers, input borders |
| Text primary | Ink | `#1A2230` | Headings and body |
| Text secondary | Slate | `#5B6676` | Labels, helper text, muted values (such as local currency) |
| Text disabled | Ash | `#9AA3B2` | Disabled controls |

**Chart colours.**
- Pay components (base / bonus / allowances): `#1E4ED8`, `#0F766E`, `#B45309`.
- Salary bands (7 ordered bands, low to high, one blue scale): `#DBE7FF`, `#B7CEFF`, `#8FB0FA`, `#6590F2`, `#3F6DE0`, `#2A52BD`, `#1B3A8F`.
- Single-series bar charts: `#1E4ED8`, with the highest bar in `#173DAE`.

Never rely on colour alone; always pair it with a label, icon or text.

## 3. Typography
Font: **Inter** (fallback: system sans-serif). Use tabular (monospaced) numerals for every number in tables and KPI cards so digits align.

| Style | Size / line height | Weight | Use |
|---|---|---|---|
| Page title | 28 / 36 px | 600 | Page headers |
| Section title | 20 / 28 px | 600 | Card and section headings |
| Subsection | 16 / 24 px | 600 | Card subtitles, dialog titles |
| Body | 14 / 20 px | 400 | Default text, table cells, inputs |
| Body strong | 14 / 20 px | 600 | Emphasis, column headers |
| Caption | 12 / 16 px | 400 | Helper text, chart axis labels, notes |
| KPI value | 32 / 40 px | 600 | Big numbers on dashboard cards |

Text is sentence case. Currency uses a thousands separator and no cents for whole dollars (for example `$45,000`, `$1.02M` in compact KPI cards).

## 4. Layout and Spacing
- 8 px spacing grid (4 px for tight gaps inside controls).
- App shell: fixed top bar (64 px, white, 1 px bottom border) with product name on the left, navigation links (Dashboard, Employees, History, Bulk update), and a user menu on the right. No sidebar.
- Content area: centred, maximum width 1280 px, 24 px page padding, 24 px gaps between cards.
- Dashboard uses a 12-column grid: 4 KPI cards (3 columns each), then 2 charts side by side (6 + 6), then full-width sections.
- Forms use a 2-column grid on desktop with 16 px gaps, single column below 900 px.
- Tables: row height 44 px, header 44 px, cell padding 12 px, numbers right-aligned.

## 5. Shape and Elevation
- Corner radius: 8 px for cards, dialogs and inputs; 6 px for buttons and chips; full pill for status chips.
- Elevation: flat. Cards use a 1 px `Line` border and no shadow. Only dialogs and menus use a soft shadow (0 8 24 rgba(26, 34, 48, 0.12)).
- Focus: 2 px `Ledger Blue` outline with 2 px offset on all interactive elements.

## 6. Component Styling (use these MUI components)
- **Buttons:** contained primary (Ledger Blue, white text) for the main action; outlined for secondary; text buttons for tertiary. Destructive actions use outlined or contained red. Height 40 px (small 32 px). Never more than one primary button per view.
- **Text fields and selects:** outlined, small size, label above the field area, 8 px radius, `Line` border, blue border on focus, red on error with helper text below.
- **Autocomplete:** outlined text field with a dropdown list; used for manager search.
- **Data table (MUI X DataGrid):** white surface, `Fog` header with 600-weight labels, 1 px row dividers, hover `#F8FAFD`, selected row `Mist Blue`, sort arrows in headers, pagination footer with rows-per-page selector (25, 50, 100). Inactive employees appear muted (Slate text) with a grey chip.
- **Status chips:** Active = green tonal (`#E6F4EA` background, `#2E7D32` text); Inactive = grey tonal (`#EEF0F4`, `#5B6676`). Change-type chips on salary history: Hire (blue tonal), Manual edit (grey tonal), Bulk update (teal tonal).
- **KPI card:** white card, 1 px border, 20 px padding; small Slate label at the top, large KPI value, optional one-line Slate caption below.
- **Cards:** white, 8 px radius, 1 px border, 20 to 24 px padding, section title at the top.
- **Radio groups:** MUI RadioGroup for choices of two or three options (bulk update: what to change, and how).
- **Stepper (bulk update):** horizontal, three steps (Target, Preview, Confirm), active step in Ledger Blue.
- **Dialogs:** 480 to 560 px wide, title, short body text, right-aligned actions (secondary then primary).
- **Alerts:** MUI Alert, standard variant, with icon; warning for organisation-wide change, error for server errors, success for completed actions.
- **Timeline (salary history):** a simple vertical line with dots (a styled list, no extra component library), newest first; each entry shows date, change-type chip, reason, and old to new values.
- **Skeletons:** grey rounded blocks that match the final layout for tables, cards and charts.
- **Empty states:** simple line icon, one-sentence explanation, one action button.
- **Snackbar:** bottom-left, for brief confirmations ("Employee saved").

## 7. Data Display and Charts
- Charts use Recharts-style simple shapes: vertical or horizontal bars, stacked bars, one stacked horizontal bar. No pie or 3D charts.
- Light grid lines (`Line`), axis labels in caption size, values on tooltips (headcount, minimum, average, maximum).
- Sort bar charts from highest to lowest. Always show the unit (USD) in the chart title or axis.
- Every chart sits in a card with a title and a one-line description of what it shows.
- Local-currency amounts are shown smaller in Slate with the label "Approximate" and the rate date, using the currency symbol and no decimals (for example `₹4,257,900`).

## 8. Content and Tone
- Plain, direct, professional. Short labels. Examples: "Add employee", "Apply to 123 employees", "No employees match your filters".
- Use realistic sample data: employees from India, the United States, the United Kingdom, Germany, Canada and Singapore; departments such as Engineering, Sales, Customer Support; job titles such as Software Engineer, Account Executive, Engineering Manager. Amounts are annual USD.
- Confirmations state exactly what will change.

## 9. Do and Don't
**Do:** keep one primary action per screen; align numbers to the right; show loading, empty and error states; keep tables as the main structure of list pages; use the blue scale consistently for ordered data.
**Don't:** use more than one accent colour outside charts; use gradients, glassmorphism, large shadows or decorative illustrations; use red for anything that is not an error or destructive; hide key numbers inside tooltips only.

## 10. Scope Rules (important)
Design only what each prompt lists. Do not invent features or decorative elements. Specifically, never add: notifications, global search, settings or help links, profile pages, avatar-menu items other than Logout, export or import buttons, dark-mode toggles, sidebars, tabs that are not listed, extra KPI cards, extra charts, extra table columns or filters, banners, footers, onboarding or promotional content. Keep exactly the navigation items given (Dashboard, Employees, History, Bulk update). When a prompt gives a count (for example four KPI cards), match it exactly.

## 11. Accessibility
- Text contrast at least 4.5:1 (body) and 3:1 (large text and UI components).
- All inputs have visible labels; error messages are text, not only colour.
- Minimum click target 32 px; visible focus ring everywhere.
- Charts have a text summary or a data table alternative.
