# TMW Operational Dashboard API — Project README

## Project Owner
- 12+ years TMW dispatcher and file maintenance experience
- DOMO administrator — mapped ETL flows and joined 6+ TMW datasets
- Google Workspace admin — enterprise domain
- GitHub: kyung83 / tmw-dashboard-api (private)
- Proven SQL access to TMWGATE and TMWGATE_Test (NOLO-SQL01)

---

## Project Purpose

Build a lightweight, secure, read-only API layer on top of TMW Suite SQL Server
to eliminate manual data transposition and power real-time operational dashboards.

All dashboards must run on a batch timer and update dynamically from TMW data.
Static snapshots are not acceptable — operational value requires near-real-time data.

---

## Database Environments

| Environment | Name | Status | Purpose |
|---|---|---|---|
| ✅ DEVELOPMENT | TMWGATE_Test | Frozen 2024 snapshot | Prove SQL structure and API plumbing |
| ✅ PRODUCTION TARGET | TMWGATE | Live — READ ONLY | Final destination after full proof |
| 🚫 NEVER | TMWGATE | Any write operation | Off limits forever |

**Server:** NOLO-SQL01.NOLO.LOCAL | **TMW Version:** 2021.1.1.2002

### TMWGATE_Test Reality
Frozen ~Nov 2024 snapshot set up by ASR/Eleos for integration testing.
Does NOT sync from production. Use for structural testing only.
Validate query logic against DOMO datasets (accounting for 24-48hr lag).

---

## Architecture

```
TMW SQL Server (TMWGATE_Test → then TMWGATE read-only)
        ↓
SQL Views / Stored Procedures (controlled access layer)
        ↓
Node.js REST API (outbound push — runs inside network)
No inbound public endpoint to SQL Server ever
        ↓
Looker Studio — primary display / dashboard layer
        +
Google Sheets — collaborative human-input fields only
```

---

## Six Target Dashboards

| # | Dashboard | Frequency | History | Priority | Automation |
|---|---|---|---|---|---|
| 1 | Load Sheet (LS) | Daily per terminal | Preserved | 🔴 First | Full |
| 2 | Shuttle Sheet | Daily (top of LS) | Preserved | 🔴 First | Full |
| 3 | OTR Fronthaul/Empty Board | Continuous | Preserved | 🟡 Second | Partial |
| 4 | OTR Backhaul Board (4/27 format) | Weekly | Preserved | 🟡 Second | High |
| 5 | LTL Empty Board | Daily | None | 🟢 Third | Medium |
| 6 | Driver Planning Sheet | Weekly | None | 🟢 Third | Partial |

### Dashboard Descriptions

**1. Load Sheet** — Daily freight assignment per cross-dock terminal. Order # → Origin → Destination → Skid Count → Driver → Trailer. 100% manual today. All fields exist in TMW.

**2. Shuttle Sheet** — Top section of Load Sheet. Dock-to-dock shuttle moves. Different data structure from main load rows. Ties into daily dock sheets. Needs separate SQL view.

**3. OTR Fronthaul/Empty Board** — Running communication board between OTR Scheduling and Backhaul dept. Shows where/when OTR trucks will be empty by day. Driver → Location → ETA → Order # → Drop Location → Notes. Planning notes at top of each day remain manual.

**4. OTR Backhaul Board** — Structured weekly planning grid paired with fronthaul board. LEFT: outbound empty trucks (city/state, driver, covered Y/N, ETA). RIGHT: inbound backhauls booked (origin → Michigan destination, driver, dispatched Y/N, order #). OTR and Backhaul boards share same underlying data — different views of same dataset.

**5. LTL Empty Board** — Daily geographic capacity snapshot. Grid organized by LTL dispatch quadrants (LOCAL, NORTH-TC/Cadillac, NORTH-Boyne/Petoskey, Tri City/Thumb, GR, PATVILLE, Dannyland, Flint Town, Lakeshore, SW, MIDDLE, West). Fields: Driver + Route Area, Empty flag, Spots Used, TKL. Not currently dynamic — automation target is batch-updated version.

**6. Driver Planning Sheet** — Weekly scheduling board by day (Sun-Sat tabs) across all terminals and driver types. Auto-populate: roster, endorsements, truck/trailer, phone. Human input retained: daily start times, trip status, confirm flags, location notes.

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

1. ✅ GitHub repo created — README.md, .env.example committed to main
2. ⬜ Add ARCHITECTURE.md, TMW_DOMAIN_KNOWLEDGE.md to main
3. ⬜ Create dev branch
4. ⬜ Identify Load Sheet + Shuttle Sheet tables from DOMO reference
5. ⬜ Write and validate SQL views in TMWGATE_Test
6. ⬜ Create tmw_api_user with scoped permissions
7. ⬜ Build first Node.js endpoint — Load Sheet JSON
8. ⬜ Test with Postman
9. ⬜ Validate against DOMO reference data
10. ⬜ Connect to Looker Studio
11. ⬜ Repeat for OTR boards (Fronthaul + Backhaul — shared view)
12. ⬜ Repeat for LTL Empty Board
13. ⬜ Repeat for Driver Planning Sheet
14. ⬜ Switch connection to TMWGATE production read-only
15. ⬜ Validate live data matches expected output

---

## What This Project Is Not
- Not SystemsLink (not purchased, not needed)
- Not a DOMO replacement
- Not a write system in Phase 1
- Not a public-facing application
- Not connected to production for writes under any circumstances
