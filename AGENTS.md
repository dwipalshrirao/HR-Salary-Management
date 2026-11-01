# AGENTS.md: rules for AI coding agents

Project: ACME Salary Management (FastAPI + SQLite backend, React (JavaScript) frontend).

## Read first
1. `docs/PRD.md` (what and why)
2. `specs/00-overview.md` (conventions, architecture, commit plan)
3. The spec for the task: `01-database-schema.md`, `02-backend-spec.md`, `03-analytics-spec.md`, `04-frontend-spec.md`, `05-seed-and-testing.md`

If a spec and the code disagree, stop and ask, or update the spec in the same commit with a reason.

## Design reference (Google Stitch via MCP)
- The UI design lives in a Stitch project, connected to OpenCode as the `stitch` MCP server. Read `design/STITCH_HANDOFF.md` before any frontend task.
- For each page, fetch the matching Stitch screen by name (for example `04 Employee detail`; there is no screen for currency rates, they are API-only) and use it as the **visual reference** only. Do not paste Stitch HTML or CSS into the app.
- Build with MUI, MUI X DataGrid and Recharts as in `specs/04-frontend-spec.md`. Take colours, type and spacing from `design/DESIGN.md` through `theme.js`.
- Precedence: specs win on behaviour, data and validation; the design wins on layout and styling. Log any difference in `design/STITCH_HANDOFF.md` section 7.
- If the Stitch MCP server is unavailable, use `design/DESIGN.md` and `design/stitch-prompts.md` and say so in the commit message.

## Working rules
- Work on **one commit-sized task at a time**, in the order in `specs/00-overview.md` sec 8 (foundation first, then vertical slices: each feature backend, then frontend). Do not start the next task.
- **Test-driven development is mandatory.** Write a failing test first, see it fail for the right reason, write the least code to pass, then refactor. Never write production code without a test that needs it. Details and the red-green-refactor loop are in `specs/05-seed-and-testing.md` sec 1a.
- Fix every bug by first writing a test that reproduces it.
- A task is not done until tests pass and `make lint` is clean.
- Keep the layering: routes (thin) -> services (rules, transactions) -> repositories (SQL).
- Money is integer cents everywhere. Never use floats for money. Frontend uses `lib/money.js`.
- Never call `date.today()` or `datetime.now()` in services; use the injected `Clock`.
- No raw string-built SQL. Use bound parameters. `sort_by` and `measure` are whitelisted, never passed through.
- Never log or return passwords, password hashes, session tokens, CSRF tokens or salary amounts in logs.
- Do not add features, endpoints or dependencies that are not in the specs. List out-of-scope items are deliberate (see PRD section 4).
- Prefer small, readable functions and clear names over cleverness. Add comments only for the "why".
- The frontend is **JavaScript** (ES modules, `.jsx` for files that contain JSX), not TypeScript. Describe shared data shapes with JSDoc `@typedef` comments.
- The API contract is `docs/openapi.json`. Do not edit it by hand; regenerate it with `make openapi` after backend changes.

## Commands
```
make setup       # install backend and frontend dependencies
make migrate     # apply Alembic migrations
make seed        # seed 10,000 employees (needs SEED_* env vars or prints a generated password)
make backend     # run API on :8000 with reload
make frontend    # run Vite on :5173 (proxies /api)
make test        # backend and frontend tests
make lint        # ruff, mypy (optional), eslint
make openapi     # export the API contract to docs/openapi.json
```

## Definition of done
- Behaviour matches the spec section named in the commit message.
- Tests written first, fast and deterministic (no network, no sleeps, fixed clock and seeds); the commit message lists them.
- No new lint warnings; no secrets or `.env` committed.
- Spec updated if behaviour changed.
- The prompt used is saved in `prompts/`.

## Commit style
Small, incremental commits, one per backend or frontend step of a slice. Format: `<type>(<feature>): <what changed> (spec <file> sec <section>)`, with a body line `Tests first: ...` listing the tests written before the code. Types: `feat`, `test`, `fix`, `chore`, `docs`.
Example: `feat(F7): bulk update preview and apply API (spec 02 sec 5.5, sec 7.3)`.
For the riskiest rules, make a `test(...)` commit first (new tests marked as expected failures), then the `feat(...)` commit that makes them pass and removes the markers.

## When unsure
Ask before guessing on: data model changes, new dependencies, anything touching auth or sessions, anything that changes an API contract.

## Prompt template for one slice
```
Read AGENTS.md and specs/00-overview.md sec 8. We are on slice S<n> (<feature>).
Do the BACKEND commit only: follow TDD (specs/05 sec 1a). First list the tests you will
write from specs/05 sec 2 and the spec section <section>. Write them, show them failing,
then implement until green, refactor, and run `make openapi`. Stop before the frontend.
```
```
Backend commit for slice S<n> is merged. Do the FRONTEND commit only: read
design/STITCH_HANDOFF.md, fetch the Stitch screen "<name>", follow specs/04 sec <section>.
Write MSW handlers and failing page tests first, then implement until green.
```
