# Architecture & Guardrails — TMW Operational Dashboard API

## Core Principle
The API is a controlled, read-only wrapper around SQL Server.
It is NOT a direct database interface.
It is NOT a replacement for TMW.
It is NOT connected to production for writes.
It is NOT a public-facing application.

---

## Hard Rules — Enforce on Every Single Response

### Database Rules
- TMWGATE_Test for all development. TMWGATE production for final read-only connection only.
- ZERO write operations (INSERT / UPDATE / DELETE) in Phase 1. If asked, refuse and flag it.
- All SQL must go through named views or stored procedures. No raw SELECT * FROM table from API.
- Do not assume column names exist. Validate against SSMS or DOMO before writing any query.
- Do not suggest schema changes to any existing TMW table or stored procedure.

### Credentials Rules
- No credentials, server names, passwords, or API keys hardcoded in any file ever.
- All sensitive values go in .env file only.
- .env is always in .gitignore — never committed to GitHub under any circumstance.
- .env.example contains placeholder values only — safe to commit.
- ⚠️ **KNOWN VIOLATION (as of Jul 2026)**: the shipped `index.html` on GitHub Pages contains `APPS_SCRIPT_URL`, `API_TOKEN`, `APP_USER`, and `APP_PASS` hardcoded in client-side JavaScript, and the repo is public. Anyone can read them and hit the Apps Script endpoint. Login is client-side and bypassable. Not blocking dashboard operation, but must be rotated + repo made private or auth moved server-side.

### Source Control Rules
- main branch = confirmed working code only.
- dev branch = all active development and testing.
- Never instruct user to push directly to main without confirming the change works first.
- Every meaningful change gets committed before moving to the next step.

### API Rules (Apps Script web app)
- Query-string token validation on every request (`?token=...`). Reject with `{error: "Unauthorized"}` if missing/wrong.
- Never expose raw SQL or Sheet error messages in responses.
- Return clean JSON errors only (`{error: "..."}`).
- One action per query (`?action=read&role=...` or `?action=refresh&role=...`).
- Redeploy required after any code change (Deploy → Manage deployments → pencil → New version → Deploy). Saving alone does not push live.

### Build Rules
- One step at a time. Confirm each step works before writing the next.
- Never write more than one build step ahead of confirmed results.
- If a query takes more than 3 seconds in SSMS it needs to be rewritten before use.

---

## System Architecture

Original vision was Node.js Express in front of SQL. Shipped implementation is different — recorded below.

```
Layer 1 — External Client
  Static index.html on GitHub Pages (kyung83.github.io/tmw-dashboard-api/)
  - Vanilla JS, no build step, no framework (React CDN referenced but not used)
  - Fetches Apps Script web app JSON with ?token=&action=read&role=
  - Sends query-string token
  - Positional COLUMNS array maps JSON array-of-arrays to named fields
  - NEVER connects directly to SQL Server or a Google Sheet
  - NEVER contains business logic beyond display/formatting

Layer 2 — Apps Script Web App
  Bound to the LTL Real Time Board Google Sheet workbook
  - doGet(e) validates ?token= then dispatches on ?action=
    - "read"    → reads the requested tab range and returns JSON rows
    - "refresh" → writes a timestamp to _Control!B{roleRow} so Python picks it up
  - Also owns applyFormatting_() which paints the sheet after Python writes
  - Redeploy after every code change (New version) or nothing goes live

Layer 3 — Python Extract Script (data transport)
  ltl_board_extract.py on NOLO-SQL01, launched via Task Scheduler "LTL Board Extract"
  - Polls _Control tab every ~30s for role refresh flags
  - Runs EXEC sp_GetLTLBoard @RoleBucket=?, @CompletedOnly=?
  - Also reads the Driver Planning workbook's Logs tab and staples
    ScheduledStart + GeotabLogin onto each driver's rows by name
  - Writes to the matching role tab in the LTL Real Time Board workbook
  - Uses pod_readonly (SQL login) + pod-pipeline-writer (Google service account)
  - Read-only against TMW — only calls the proc

Layer 4 — SQL Access Layer
  Stored Procedures (parameterized, read-only)
  - sp_GetLTLBoard (@RoleBucket, @CompletedOnly) — 9-CTE proc, 29 output columns
  - Naming convention going forward: sp_Get{Board}[{Filter}] (e.g. sp_GetLoadSheetByTerminal for future dashboards)
  - This layer is the safety boundary — no ad-hoc SELECTs from the extract script

Layer 5 — Database
  Development: TMWGATE_Test (live ASR/Eleos synthetic fixtures — structure/vocab only, never volumes)
  Production:  TMWGATE (read-only SELECT / EXEC via pod_readonly)
```

---

## Coordination & Deployment Discipline (lessons from Jul 8 2026)

The pipeline has 5 layers and touching any one of them can affect the others.
These rules exist because they've all been violated at least once already:

### Multi-editor coordination on `index.html`
- **Two people editing the same file in parallel is how work gets erased.** On Jul 8, a local git push at 10:05 (header button + scroll preservation) was overwritten 32 minutes later by a GitHub-web "Upload files" that carried a stale local copy. Git recorded it as one commit that both added a new feature AND wiped the earlier work.
- **Rule: always `git pull` before editing `index.html`.** Doesn't matter if you're using local git, the GitHub web editor, or the "Upload files" button — pull first, or you might be starting from a stale copy.
- **Rule: `git status` + `git log --oneline -3` before any edit session.** If your local's HEAD isn't `origin/main`'s HEAD, stop and reconcile.
- The GitHub web "Upload files" flow does NOT warn about overwriting newer content. It just replaces the file. Treat it as destructive.
- Commits made via the GitHub web UI show `committer: GitHub <noreply@github.com>` in `git log --format=fuller`. Commits via local `git push` show your identity. Useful diagnostic when investigating a mystery.

### Adding a new column end-to-end (SQL → Python → Sheet → Apps Script → HTML)
Order matters. Do them in this sequence or the dashboard breaks:

1. **SQL proc** — add the column to `sp_GetLTLBoard`'s SELECT.
2. **Python (`ltl_board_extract.py`)** — usually no change needed (writes whatever the proc returns). Verify the clear-range covers the new column (widen `A4:X` → `A4:AD` etc. if needed).
3. **Apps Script `applyFormatting_`** — widen `sheet.getRange(5, 1, lastRow - 4, N).getValues()` to include the new column count.
4. **Apps Script `doGet` (`action === "read"`)** — widen `sheet.getRange(5, 1, lastRow - 4, N).getValues()` here **too** (same number). This is the one that actually reaches the browser. Missing this is the #1 gotcha.
5. **Deploy the Apps Script** — Deploy → Manage deployments → pencil on active → Version: **New version** → Deploy. Saving alone changes nothing on the live URL.
6. **`index.html` `COLUMNS` array** — append the new field name in the **exact same position** the sheet has it. `COLUMNS` is positional — it maps array indices to named fields. If the sheet has 29 columns and you list 28, the last real value is silently dropped.
7. **`index.html` `parseRows`** — capture `rec.NewField` on the driver object.
8. **`index.html` `renderBoard`** — render it wherever it belongs.
9. **Commit + push** `index.html`. Fastly cache invalidates in ~15-30 seconds.

### Apps Script save vs deploy
- **Save (Ctrl+S) does NOT push to the live URL.** It updates the editor draft only.
- The live web app runs whatever version was last **deployed**, not what you last saved.
- To actually push a change: Deploy → **Manage** deployments (not "New deployment", which creates a second web app with a different URL) → pencil icon → Version dropdown = **New version** → Deploy.
- After deploy, verify by hitting the web app URL directly and checking the response — don't trust that "Deployment successfully updated" means what you wanted actually took effect.

### The Apps Script "same number in two places" trap
The Apps Script has `sheet.getRange(5, 1, lastRow - 4, N).getValues()` in two functions:
- `applyFormatting_` (line ~70 in current file) — used by the sheet formatter
- `doGet` inside `if (action === "read")` — used by the dashboard

Both `N`s must match the sheet's actual column count. Changing only one leaves the other silently truncating data. On Jul 8 the `doGet` copy stayed at `26` for several minutes after `applyFormatting_` was updated to `29`, and the dashboard chips were blank the whole time.

### `COLUMNS` array is positional, not by name
```javascript
const COLUMNS = ["RoleBucket", "EquipClass", ..., "GeotabLogin"];
COLUMNS.forEach((col, i) => rec[col] = row[i] || "");
```
The array's **order** determines which JSON index maps to which field name. If the sheet reorders columns or you insert a new one, you MUST insert into `COLUMNS` at the same position — not at the end. Otherwise every field after that position gets misaligned.

### JSON transport shape
The Apps Script `doGet` returns:
```json
{ "rows": [ [col0, col1, col2, ...], [col0, col1, col2, ...], ... ] }
```
Array of arrays, not array of objects. Field names live only in the `index.html` `COLUMNS` array — they never leave the browser.

### Fastly / GitHub Pages caching
- GitHub Pages sits behind Fastly. Cache TTL on `index.html` is ~10 minutes but Fastly often invalidates faster on push.
- Two browsers on the same URL can serve **different bytes** for the same seconds after a push. This is normal, not a bug.
- Hard-refresh (Ctrl+F5) usually forces a fresh HTML fetch. If it doesn't, close the tab entirely and reopen — the JS engine's in-memory state can persist across Ctrl+F5.
- Cache-buster query strings (`?_cb=<timestamp>`) force a bypass at both Fastly and the browser cache. Useful when debugging.
- After a push, wait 15-30 seconds before assuming "the live page still shows the old version" is a real problem.

### The pod-pipeline-writer service account
- Google service account `pod-pipeline-writer@emerald-rhythm-500213-g2.iam.gserviceaccount.com` owns all writes to the LTL Real Time Board workbook.
- **Also needs Viewer access on the Driver Planning workbook** so the Python can read the `Logs` tab for `ScheduledStart` + `GeotabLogin` join. If chips come back blank across the board, first thing to check is whether the service account is still shared on the planning workbook.
- If shared drops or a new dashboard adds a new source workbook, share the service account (Viewer for reads, Editor for writes) before expecting data to appear.

### Rendering containment on large boards (`content-visibility: auto`)
The LTL Trip Folder Board renders every active driver at once — ~110 driver blocks × ~5-8 stops each = ~750 table rows and ~16K DOM nodes in a single page. Without rendering containment, Chrome must include every driver block in every layout and paint pass. Toggling `display: none` on a single `.stops-wrap` then forces Chrome to invalidate paint for the entire vertical stack below it. On a workstation under memory pressure (Citrix + Chrome tabs collectively holding several GB), this compounds with Windows swap-in latency and produces a 10-20s freeze that affects other Chrome processes as well (not just the tab).

**This is not a JS bug and cannot be diagnosed by profiling the click handler.** Programmatic and real-click toggles both measure 2-5ms in the handler itself. The freeze happens after the JS returns, in Chrome's async paint/composite phase.

**Fix (shipped Jul 8 2026):** `.driver-block` in `index.html` carries:
```css
content-visibility: auto;
contain-intrinsic-size: auto 60px;
```
`content-visibility: auto` lets Chrome skip style/layout/paint for off-screen blocks. `contain-intrinsic-size: auto 60px` reserves a placeholder height so scroll position stays stable and Chrome remembers the real size after first render. Ctrl+F browser find still traverses skipped content (Chrome 90+), and the `applySearch()` filter still works because it rebuilds the DOM from JS state rather than walking hidden nodes.

**Rule for future dashboards:** any board that renders repeated blocks where N could hit triple digits (LTL Empty Board quadrant grid, OTR Fronthaul/Backhaul boards, Driver Planning Sheet rebuild) gets `content-visibility: auto` + a `contain-intrinsic-size` estimate on the repeated block class from day one. Cheap insurance. Do not wait until users report freezing.

**Diagnostic trap to avoid:** when a user reports a freeze that "also lags the Claude extension" or "freezes the whole PC", that is the tell that the stall is browser-wide (paint/composite or OS-level), not a per-tab JS main-thread stall. Chrome's built-in Task Manager (Shift+Esc) separates tab processes from extension processes and confirms this quickly.

### Sticky sub-header nav + anchor-jump landing
The LTL Trip Folder Board's tab bar (`LTL / Flatbed / OTR / Completed Trips`) doubles as a jump-nav row — terminal name buttons on the right side smooth-scroll to each terminal section. Sticky positioning + anchor-jump has three gotchas worth documenting because they'll come back on every future dashboard that grows beyond a single viewport.

**Layered sticky bars need explicit z-index.** The NORLO header is `position: sticky; top: 0; z-index: 100`. The tab bar sits directly below at `position: sticky; top: 56px; z-index: 99`. Both stay pinned as the driver rows scroll under them. NORLO's higher z-index means it wins any overlap fight — critical because dropdowns, modals, and the HOS backdrop (`z-index: 200`) all need to layer sensibly against the stack.

**Anchor jumps require `scroll-margin-top` or targets land behind sticky bars.** `element.scrollIntoView({block: 'start'})` scrolls the element to y=0 in the viewport, which for a two-tier sticky header stack means the target lands *behind* both bars. Fix: `.terminal-block { scroll-margin-top: 110px; }` (56px NORLO + ~44px tab bar + 10px breathing room). The browser then offsets the scroll so the target lands just below the sticky stack. Applies wherever the dashboard uses smooth-scroll to an anchor — future in-page nav in the OTR boards or Driver Planning rebuild needs the same treatment.

**Dynamic nav buttons must rebuild on every render, not once at load.** The jump buttons in `#terminalJumpBar` are populated inside `renderBoard()` from the same `rendered` array the terminal blocks come from. This gives you three behaviors for free: (1) tab switch swaps the button set to the new tab's terminals, (2) search filter narrows the buttons alongside the driver blocks (a Grand Rapids driver name shown → only Grand Rapids button), (3) empty state clears the button bar. Do not attach them at load time or hardcode the terminal list — the whole point of the pattern is that the nav matches what's on screen.

**Floating action buttons use scroll listener + `.visible` class.** The back-to-top pill (`#backToTop`) is a fixed-position element hidden by `display: none` at load. A passive scroll listener toggles `.visible { display: block }` past `window.scrollY > 400`. Two rules: (1) always attach with `{ passive: true }` so it doesn't block scroll performance (matters at 60fps on long documents), (2) don't animate the show/hide — keep it a display swap. Fade-in on a document-wide scroll listener is a subtle performance tax that adds up on long boards.

**Rule for future dashboards:** any board that grows past a single viewport (LTL Empty Board, OTR Fronthaul/Backhaul, Driver Planning Sheet rebuild) gets this pattern from day one — sticky sub-header with explicit z-index below the NORLO header, `scroll-margin-top` on major anchor targets, dynamic in-nav buttons rebuilt on every render, floating back-to-top past 400px. Same green pill vocabulary (`#052e16` bg, `#16a34a` border, `#86efac` text, mono uppercase) keeps the nav language consistent across dashboards.

---

## SQL User Setup

Actual shipped setup for the LTL Trip Folder Board uses `pod_readonly`
(carried over from the POD Quality Checker project). Password lives in the
Python extract script's environment on NOLO-SQL01, not in this repo.

```sql
-- Already exists in TMWGATE production (created for the POD project).
-- Only new grant needed for the LTL Board was:
GRANT EXECUTE ON dbo.sp_GetLTLBoard TO pod_readonly;
```

For future dashboards, grant only what that dashboard needs. Never `db_datareader` (too broad).
Naming convention for new procs: `sp_Get{BoardName}[{Filter}]`.

---

## View vs Stored Procedure

| Use a VIEW when | Use a STORED PROCEDURE when |
|---|---|
| Simple SELECT, no filters needed | Parameters needed (terminal, date range, etc.) |
| Full dataset returned as-is | Business logic or filtering required |
| No user input | Future write operations may be added |

---

## .env File Structure
Never hardcode. Always reference process.env.[VARIABLE]

```
DB_SERVER=
DB_NAME=
DB_USER=
DB_PASSWORD=
API_KEY=
PORT=
```

See .env.example for this structure with placeholder values.

---

## .gitignore Requirements

Must include at minimum:
```
.env
node_modules/
*.log
```

---

## TMW-Specific Safety Rules
Proven from direct SQL investigation — non-negotiable

- DO NOT modify cmp_terms — live freight billing code, active across all company records
- DO NOT repurpose THR — active legacy code in billing/costing stored procedures
- DO NOT touch SettlementPaymentTerms or BillingOutputTerm — net terms project only
- DO NOT query invoiceheader or financial tables unless explicitly required for a specific dashboard
- DO NOT expose driver phone numbers or PII in any endpoint without explicit requirement
- DO NOT suggest any change to existing TMW stored procedures or views

---

## Data Validation Rule

Every query must be validated against DOMO reference data before use in any endpoint.
If results do not match DOMO (accounting for 24-48hr lag), fix the query.
Never adjust the validation to match a bad query.

---

## Error Handling Standard

**Apps Script `doGet` (returns JSON):**

```javascript
if (token !== validToken) {
  return ContentService.createTextOutput(JSON.stringify({error: "Unauthorized"}))
    .setMimeType(ContentService.MimeType.JSON);
}
// ... never surface SQL/Sheet internals to the browser
```

**Python extract loop (writes status back to _Control):**

```python
try:
    # fetch_board / write_board / etc.
    set_status(client, role, "COMPLETE")
except Exception as exc:
    log(f"ERROR during {role} refresh: {exc}")
    set_status(client, role, "ERROR", detail=str(exc)[:60])
```

Errors are logged internally and surface to dispatchers only as short status text on the sheet — never as raw exceptions.

---

## Full Stop Triggers
Stop and flag before proceeding if any of the following come up:

- Any suggestion to connect to TMWGATE production for writes
- Any raw table query from the API layer
- Any INSERT / UPDATE / DELETE operation
- Any hardcoded credential or server name in code
- Any suggestion to push credentials to GitHub
- Any endpoint exposing PII without explicit requirement
- Any modification to existing TMW tables, views, or stored procedures
- Any suggestion to use db_datareader or broad permission grants
- Any suggestion to open an inbound port to SQL Server

---

## Future Phases — Do Not Build Now

- Write-back of dispatcher data into TMW (Phase 2 — separate project)
- OAuth or role-based access beyond API key
- Real-time websocket connections
- Multi-terminal role isolation
