# TMW Operational Dashboard API — Project README

## Project Owner
- 12+ years TMW dispatcher and file maintenance experience
- DOMO administrator — mapped ETL flows and joined 6+ TMW datasets
- Google Workspace admin — enterprise domain
- GitHub: kyung83 / tmw-dashboard-api (public — hosts GitHub Pages dashboard; see ARCHITECTURE.md security note re: hardcoded client-side token)
- Proven SQL access to TMWGATE and TMWGATE_Test (NOLO-SQL01)

---

## Project Purpose

Build a lightweight, secure, read-only API layer on top of TMW Suite SQL Server
to eliminate manual data transposition and power real-time operational dashboards.

All dashboards must run on a batch timer and update dynamically from TMW data.
Static snapshots are not acceptable — operational value requires near-real-time data.

---

## Systems & Naming (read this first, everyone confuses these)

Three distinct systems have "LTL" or "board" in their name. They are NOT interchangeable:

| Nickname | What it actually is | Where it lives |
|---|---|---|
| **LTL Real Time Board** (workbook) | Google Sheets workbook the Python writes to. Tabs: `LTL`, `Flatbed`, `OTR`, `Completed Trip`, `Unclassified`, `_Control`, `_Notes` | Google Drive |
| **Driver Planning workbook** | Separate Google Sheets workbook. Weekly scheduling grid + a `Logs` tab that Python reads to staple `ScheduledStart` and `GeotabLogin` onto each driver row | Google Drive |
| **The dashboard** / **index.html** / **the GitHub Pages site** | Static HTML at kyung83.github.io/tmw-dashboard-api/ that dispatchers actually look at | GitHub Pages (main branch root) |

Rules:
- "Update the sheet" is ambiguous — always name which workbook.
- "Update the board" without context usually means `index.html`, but confirm.
- "TMWGATE" is the SQL database. "NOLO-SQL01" is the Windows server it runs on. Not the same thing — you don't need SSMS open to work on the VM's Python script.

---

## Database Environments

| Environment | Name | Status | Purpose |
|---|---|---|---|
| ✅ DEVELOPMENT | TMWGATE_Test | Live w/ ASR/Eleos synthetic fixtures — structure/vocab only, never trust volumes | Prove SQL structure and API plumbing |
| ✅ PRODUCTION TARGET | TMWGATE | Live — READ ONLY | Final destination after full proof |
| 🚫 NEVER | TMWGATE | Any write operation | Off limits forever |

**Server:** NOLO-SQL01.NOLO.LOCAL | **TMW Version:** 2021.1.1.2002

### TMWGATE_Test Reality
NOT a frozen snapshot. Live environment receiving synthetic integration fixtures from ASR/Eleos,
dated through current sessions (dummy drivers such as STIST "Stinky Steve", ONEMO "Mock One" observed).
Trust structure and vocabulary only — never trust volumes or business meaning.
Does NOT sync from production.
Validate query logic against DOMO datasets (accounting for 24-48hr lag).

---

## Architecture

Original vision (not built): Node.js REST API + Looker Studio.
Actual shipped pipeline for the LTL Trip Folder Board:

```
TMW SQL Server (TMWGATE, read-only via pod_readonly login)
        ↓ EXEC sp_GetLTLBoard @RoleBucket, @CompletedOnly
Python script on NOLO-SQL01 (ltl_board_extract.py, runs continuously via Task Scheduler)
        ↓ writes 29 columns to Google Sheet
LTL Real Time Board workbook (Google Sheets; pod-pipeline-writer service account)
        ↓ Apps Script doGet(?token=...&action=read&role=...) returns JSON
Static index.html on GitHub Pages (main branch root; kyung83.github.io/tmw-dashboard-api/)
        ↓
Dispatchers' browsers (LTL / Flatbed / OTR / Completed Trips tabs)
```

No inbound public endpoint to SQL Server. No Node.js layer.
Looker Studio not in use.

---

## Target Dashboards

| # | Dashboard | Frequency | History | Status |
|---|---|---|---|---|
| 0 | **LTL Trip Folder Board** (real-time shot-clock for all active local drivers) | Live, on-demand + auto-refresh | Not preserved | ✅ SHIPPED (Jul 2026) — this is what dispatchers use today |
| 1 | Load Sheet (LS) | Daily per terminal | Preserved | ⏸ DEFERRED — DOMO field confirmations still open |
| 2 | Shuttle Sheet | Daily (top of LS) | Preserved | ⏸ DEFERRED (paired with Load Sheet) |
| 3 | OTR Fronthaul/Empty Board | Continuous | Preserved | 🟡 PLANNED |
| 4 | OTR Backhaul Board (4/27 format) | Weekly | Preserved | 🟡 PLANNED |
| 5 | LTL Empty Board (geographic quadrant snapshot — see §5 below; DIFFERENT from #0 above) | Daily | None | 🟢 PLANNED |
| 6 | Driver Planning Sheet | Weekly | None | 🟢 PLANNED |

### Dashboard Descriptions

**0. LTL Trip Folder Board (SHIPPED)** — Real-time shot-clock replicating the TMW trip folder bottom grid for all active local drivers, grouped by terminal, across three role-scoped tabs (LTL / Flatbed / OTR) plus a Completed Trips tab. Backed by `sp_GetLTLBoard` in TMWGATE. Served via `ltl_board_extract.py` (on NOLO-SQL01, Task Scheduler) → Google Sheet → Apps Script `doGet` → `index.html` on GitHub Pages. Login-vs-scheduled-start chips join from a Logs tab in the Driver Planning workbook by driver name (server-side, in Python).

**1. Load Sheet** — Daily freight assignment per cross-dock terminal. Order # → Origin → Destination → Skid Count → Driver → Trailer. 100% manual today. All fields exist in TMW.

**2. Shuttle Sheet** — Top section of Load Sheet. Dock-to-dock shuttle moves. Different data structure from main load rows. Ties into daily dock sheets. Needs separate SQL view.

**3. OTR Fronthaul/Empty Board** — Running communication board between OTR Scheduling and Backhaul dept. Shows where/when OTR trucks will be empty by day. Driver → Location → ETA → Order # → Drop Location → Notes. Planning notes at top of each day remain manual.

**4. OTR Backhaul Board** — Structured weekly planning grid paired with fronthaul board. LEFT: outbound empty trucks (city/state, driver, covered Y/N, ETA). RIGHT: inbound backhauls booked (origin → Michigan destination, driver, dispatched Y/N, order #). OTR and Backhaul boards share same underlying data — different views of same dataset.

**5. LTL Empty Board** — Daily geographic capacity snapshot. Grid organized by LTL dispatch quadrants (LOCAL, NORTH-TC/Cadillac, NORTH-Boyne/Petoskey, Tri City/Thumb, GR, PATVILLE, Dannyland, Flint Town, Lakeshore, SW, MIDDLE, West). Fields: Driver + Route Area, Empty flag, Spots Used, TKL. Not currently dynamic — automation target is batch-updated version. NOT the same thing as the LTL Trip Folder Board (#0 above) — this one is a capacity/geographic view, not a per-driver trip shot-clock.

**6. Driver Planning Sheet** — Weekly scheduling board by day (Sun-Sat tabs) across all terminals and driver types. Auto-populate: roster, endorsements, truck/trailer, phone. Human input retained: daily start times, trip status, confirm flags, location notes. Currently the source of the Logs tab that the LTL Trip Folder Board joins for login times.

---

## Backhaul Business Logic (Critical)

- All backhauls return to Michigan — no out-of-state backhauls for dry van fleet
- High volume of internal Michigan customers requires constant inbound truck availability
- OTR cycle: Leave Michigan loaded → empty at destination → backhaul to Michigan → turn back out
- Backhaul dept needs: WHERE truck empty + WHEN (ETA)
- OTR scheduling needs: WHEN truck back in Michigan to plan next outbound

---

## Source Control

- **Repo:** Private — kyung83/tmw-dashboard-api
- **main branch:** Working confirmed code only — never commit directly
- **dev branch:** All active development and testing
- **Credentials:** Never in GitHub — .env file only, always in .gitignore
- **.env.example:** Safe to commit — placeholder values only

---

## Build Order

**LTL Trip Folder Board (SHIPPED, Jul 2026):**
1. ✅ GitHub repo created — README.md, .env.example committed to main
2. ✅ ARCHITECTURE.md, TMW_DOMAIN_KNOWLEDGE.md committed to main
3. ✅ dev branch created
4. ✅ Validated TMWGATE schema via INFORMATION_SCHEMA (structure/vocab only)
5. ✅ Built and deployed `sp_GetLTLBoard` (9-CTE proc, `@RoleBucket`, `@CompletedOnly` params)
6. ✅ Created LTL Real Time Board Google Sheet workbook (tabs: LTL, Flatbed, OTR, Completed Trip, Unclassified, _Control, _Notes)
7. ✅ Deployed `ltl_board_extract.py` on NOLO-SQL01 via Task Scheduler (uses pod_readonly SQL login + pod-pipeline-writer service account)
8. ✅ Deployed Apps Script bound to the LTL Real Time Board workbook (`doGet` serves JSON, `applyFormatting_` paints the sheet, menu buttons trigger refresh)
9. ✅ Deployed static `index.html` on GitHub Pages (main branch root) with token-based auth against the Apps Script web app
10. ✅ Joined driver login times server-side in Python from the Driver Planning workbook's Logs tab (ScheduledStart + GeotabLogin)
11. ✅ Added Completed Trips tab (SQL `@CompletedOnly` + frontend rendering)
12. ✅ Added HOS chip per driver — Apps Script `action=hos&email=X` calls Geotab, `index.html` opens modal with Drive Time Remaining + Workday Remaining (Jul 8 2026)
13. ✅ Applied `content-visibility: auto` + `contain-intrinsic-size: auto 60px` to `.driver-block` in `index.html` to eliminate collapse/expand freeze on the 110-driver DOM (Jul 8 2026 — see ARCHITECTURE.md for the paint/composite lesson)

**Deferred / Not started:**
- ⏸ Load Sheet & Shuttle Sheet dashboards (DOMO field confirmations still open)
- 🟡 OTR Fronthaul/Backhaul boards
- 🟢 LTL Empty Board (geographic quadrants — distinct from Trip Folder Board)
- 🟢 Driver Planning Sheet dashboard rebuild

**Any future dashboard should follow the SHIPPED pipeline shape above** (SQL proc → Python on VM → Google Sheet → Apps Script `doGet` → static HTML on GitHub Pages), not the original Node.js/Looker Studio vision.

---

## What This Project Is Not
- Not SystemsLink (not purchased, not needed)
- Not a DOMO replacement
- Not a write system in Phase 1
- Not a public-facing application
- Not connected to production for writes under any circumstances
