# TMW Operational Dashboard API — Project README

## Project Owner
- 12+ years TMW dispatcher and file maintenance experience
- DOMO administrator — has mapped ETL flows and joined 6+ TMW datasets
- Google Workspace admin — enterprise domain
- GitHub account: kyung83
- Proven SQL access to TMWGATE and TMWGATE_Test (NOLO-SQL01)

---

## Project Purpose

Build a lightweight, secure, read-only API layer on top of the TMW Suite SQL Server
database to eliminate manual data transposition and power real-time operational dashboards.

Target dashboards replace three core operational documents rebuilt manually every day:

1. **Load Sheet (LS)** — Rebuilt every morning per cross-dock terminal. Driver → Order → Origin → Destination → Skid Count.
2. **OTR Empty Board** — Weekly outbound/inbound load coverage planning. Driver coverage, order numbers, appointment times.
3. **Driver Planning Sheet** — Weekly driver scheduling board by day. 300+ drivers across 14+ locations with start times, trip status, truck/trailer, endorsements.

---

## Database Environments — CRITICAL

| Environment | Name | Status | Purpose |
|---|---|---|---|
| ✅ USE FOR DEVELOPMENT | TMWGATE_Test | Frozen 2024 snapshot | Prove SQL structure, views, API plumbing |
| ✅ USE FOR PRODUCTION DASHBOARDS | TMWGATE | Live production — READ ONLY | Final destination after TMWGATE_Test proven |
| 🚫 NEVER | TMWGATE | Any write operation | Off limits forever in this project |

**Server:** NOLO-SQL01.NOLO.LOCAL
**SQL Server Version:** 14.0.3525.1
**TMW Version:** 2021.1.1.2002
**Access:** Remote Desktop / Citrix → SSMS

### TMWGATE_Test — Important Context
TMWGATE_Test is a **frozen snapshot from approximately late 2024**.
It was originally set up by ASR/Eleos for their integration sandbox testing.
It does NOT sync from production in real time.

**How to use it correctly:**
- Use it to prove SQL view structure and API plumbing work
- Use DOMO datasets (accounting for 24-48hr lag) to validate query logic
- Use it to prove authentication and endpoint behavior
- Do NOT expect current data — the data is old and that is fine for structural testing

### TMWGATE Production — Read Only
Once SQL views and API are proven in TMWGATE_Test:
- Point the production API connection at TMWGATE
- Read-only SELECT access only — enforced at SQL user permission level
- All access still via named views or stored procedures only
- No exposed public endpoints — outbound push architecture only

---

## Architecture

```
TMW SQL Server (TMWGATE_Test for dev / TMWGATE for production)
        ↓
SQL Views / Stored Procedures (controlled access layer)
        ↓
Lightweight REST API (Node.js / Express)
        ↓
Outbound script — runs INSIDE the network
No inbound public endpoint to SQL Server ever
        ↓
Looker Studio (primary display / dashboard layer)
        +
Google Sheets (collaborative human-input layer only)
```

### Why Outbound Not Inbound
Google Apps Script runs on Google's servers and would require an inbound public
endpoint to reach SQL Server — creating an attack surface on the database layer.
A script running inside the network connects to SQL Server locally and pushes
data outbound via HTTPS. No ports opened. No SQL Server exposed externally.

---

## Source Control

- **Repository:** Private GitHub repo (kyung83)
- **Rule:** `main` branch = working proven code only
- **Rule:** All changes developed and tested in `dev` branch first
- **Rule:** Never merge `dev` into `main` until the change is confirmed working
- **Credentials:** NEVER committed to GitHub under any circumstances
- **Sensitive values:** All in `.env` file which is listed in `.gitignore`
- See `.env.example` for credential structure with placeholder values only

---

## Data Source Priority

1. **TMWGATE_Test** — prove structure and plumbing first
2. **DOMO dataset maps** — validate query logic against known good data (account for lag)
3. **TMWGATE production** — final read-only destination after full proof of concept

---

## Phase 1 — Read Only (current scope)

- No INSERT / UPDATE / DELETE under any circumstances
- No write operations to any TMW database
- All SQL access via views or stored procedures — no raw table queries from API
- Dedicated read-only SQL user (tmw_api_user) with minimal scoped permissions
- API key authentication on all endpoints
- Private GitHub repo with .env excluded from all commits

## Phase 2 — Future (not in scope, not designed yet)

- Controlled write-back of dispatcher-transformed data into TMW
- Must be fully designed and safety-reviewed as a completely separate project
- Will require ASR/vendor coordination before touching any production write logic

---

## Build Order

1. Create private GitHub repo — add .gitignore and .env.example first commit
2. Confirm TMWGATE_Test table structure matches TMWGATE for target tables
3. Identify Load Sheet tables from DOMO reference maps
4. Write and validate SQL view in TMWGATE_Test
5. Create tmw_api_user with scoped permissions in TMWGATE_Test
6. Build first Node.js API endpoint returning Load Sheet data as JSON
7. Test with Postman against TMWGATE_Test
8. Validate data shape against DOMO reference
9. Connect to Looker Studio
10. Repeat pattern for OTR Board
11. Repeat pattern for Driver Planning Sheet
12. Switch connection to TMWGATE production read-only
13. Validate live data matches expected output

---

## Operational Context

- 300+ drivers across 14+ terminal locations
- Driver types: OTR, Local, Part-Time, Box Truck, Team
- Terminals: Clare, Cadillac, GR, Boyne, Sussex WI, Ludington, Fort Wayne, Flint
- Endorsements: TANKER, ENHANCED TANKER, HAZ, ENHANCED HAZ TANKER, TWIC
- Load types: LTL, OTR, BT (Bobtail), TKL, Team

---

## What This Project Is Not

- Not a SystemsLink implementation (not purchased, not needed)
- Not a DOMO replacement
- Not a write system in Phase 1
- Not a public-facing application
- Not connected to production for writes under any circumstances
