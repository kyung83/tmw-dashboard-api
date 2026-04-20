# TMW Domain Knowledge — Operational Reference

## Purpose
This file prevents re-learning TMW context mid-project.
All facts confirmed from direct SQL investigation of NOLO-SQL01
and analysis of actual operational working documents.

---

## Environment Facts (Confirmed)

| Item | Value |
|---|---|
| Server | NOLO-SQL01.NOLO.LOCAL |
| SQL Server Version | 14.0.3525.1 |
| TMW Version | 2021.1.1.2002 |
| Production DB | TMWGATE |
| Test DB | TMWGATE_Test |
| Test DB Status | Frozen snapshot ~Nov 2024 — set up by ASR/Eleos for integration testing |
| Test DB Sync | Does NOT sync from production |
| Access Method | Remote Desktop / Citrix → SSMS |

---

## Key Tables (Confirmed to Exist)

| Table | Purpose | Notes |
|---|---|---|
| orderheader | Core order/load records | Primary table for all dashboards |
| company | Customer and company master | cmp_terms and cmp_Payment_terms live here |
| invoiceheader | Invoice records | Financial — handle carefully |
| invoiceselection | Invoice output config | Static terms text — not logic |
| SettlementPaymentTerms | Terms definition | EXISTS but EMPTY — net terms project only |
| BillingOutputTerm | Invoice output terms | EXISTS but EMPTY — net terms project only |
| Billto_Control | Bill-to config | fgt_terms lives here |
| driver | Driver master | Confirm column names before use |

---

## Critical Column Facts

- `company.cmp_terms` — freight billing responsibility (THR, PRE, COL) — DO NOT USE FOR PAYMENT TERMS
- `company.cmp_Payment_terms` — payment terms field — currently NULL — net terms project only
- `invoiceheader.ivh_GPDuedate` — currently NULL — no active due date engine
- `invoiceheader.ivh_terms` — carries term code only — no days mapping

## THR Warning
THR is active in company.cmp_terms, invoiceheader.ivh_terms, and usp_getordercosting.
DO NOT modify, repurpose, or reference THR in this project.

---

## Operational Domain Knowledge

### Company Structure
- 300+ drivers across 14+ terminal locations
- Two primary dispatch departments: OTR Scheduling and Backhaul Coordination
- These two departments work in tandem and require shared visibility

### Terminal Locations
Clare, Cadillac, Grand Rapids (GR), Boyne, Sussex WI, Ludington, Fort Wayne, Flint
(validate full list against driver table)

### Driver Types
OTR, Local, Part-Time, Box Truck, Team

### Load Types
LTL, OTR, BT (Bobtail), TKL, Team

### Endorsements
TANKER, ENHANCED TANKER, HAZ, ENHANCED HAZ TANKER, TWIC Card, None

### Restrictions
Automatic transmission, Under 21, None

### Backhaul Logic (Critical for OTR Dashboards)
- All backhauls return to Michigan — no backhauls booked going to other states
- Reason: high volume of internal Michigan customers requiring outbound loads
- OTR trucks leave Michigan with a load, go empty at destination, get a backhaul back to Michigan, then turn back out
- Backhaul sources: spot boards, 3PLs, and internal customers
- Backhaul coordinators need: WHERE the truck will be empty + WHEN (ETA)
- OTR schedulers need: WHEN the truck returns to Michigan to plan next outbound load

---

## Six Target Dashboards (Complete List)

### 1. Load Sheet — LS (Highest Priority — Start Here)
**Purpose:** Daily freight assignment board per terminal cross-dock location
**Frequency:** Rebuilt every morning per terminal
**History:** Historical copies preserved
**Structure:** One row per order
**Key Fields:** Order #, Origin, Destination, Skid Count, Driver, Trailer, Notes
**Current Pain:** 100% manual data entry from TMW screens every morning
**Automation Target:** Full — all fields exist in TMW orderheader and driver tables
**Notes:** Top section contains Shuttle loads (see Shuttle Sheet below)

### 2. Shuttle Sheet
**Purpose:** Top section of the LS Load Sheet — shuttle/dock-to-dock moves
**Frequency:** Daily — part of Load Sheet rebuild
**Relationship:** Embedded in Load Sheet but structurally different
**Data Structure:** Different from main load rows — shorter moves, dock-to-dock
**Ties Into:** Daily dock sheets for each cross-dock location
**Automation Target:** Needs separate SQL view from main Load Sheet
**Status:** Structure needs further investigation before building view

### 3. OTR Fronthaul / Empty Board (OTR Sheet)
**Purpose:** Running communication board between OTR Scheduling and Backhaul dept
**Frequency:** Updated continuously throughout the week
**History:** Historical copies preserved (dated tabs in workbook)
**Structure:** Day-of-week sections, one row per driver showing empty status
**Key Fields:** Driver, Current Location (where empty), ETA, Order #, Drop Location, Notes
**Also Contains:** Planning notes and routing decisions at top of each day section
**Current Pain:** Manual updates as dispatchers get info from drivers
**Automation Target:** Partial — location/ETA/order# from TMW, planning notes remain manual
**Relationship to OTR 4/27:** Same operational need, different view format (see below)

### 4. OTR Backhaul Board (OTR 4/27 Sheet)
**Purpose:** Structured weekly planning board — pairs outbound empty trucks with inbound backhauls
**Frequency:** Weekly — organized by specific date (Monday 4/27, Tuesday 4/28 etc.)
**History:** Historical copies preserved (dated tabs)
**Structure:** Split left/right
  - LEFT (Outbound/Empty): City/State where truck will be empty, Driver, Covered Y/N, APPT time, ETA
  - RIGHT (Inbound/Backhaul): Origin to Michigan destination, Driver, Dispatched Y/N, Order #, Date
**Current Pain:** Manual coordination between OTR schedulers and backhaul coordinators
**Automation Target:** High — city/state empty location, ETA, order# all in TMW
**Relationship to OTR Sheet:** Same data, different view — OTR Sheet is the running communication log,
OTR 4/27 is the structured planning format. Both should eventually pull from same SQL view.

### 5. LTL Empty Board (LTL Sheet)
**Purpose:** Daily geographic capacity snapshot for Local LTL drivers
**Frequency:** Rebuilt daily — NO historical copies preserved
**Structure:** Grid organized by geographic quadrants defined by LTL dispatch team
**Quadrants:**
  - LOCAL (Clare area)
  - NORTH — Traverse City / Cadillac
  - NORTH — Boyne City / Petoskey
  - Tri City / Thumb
  - GR (Grand Rapids)
  - PATVILLE
  - Dannyland
  - Flint Town
  - Lakeshore
  - SW (Southwest)
  - MIDDLE
  - West
**Key Fields per Driver:** Driver & Area route, Empty flag (x = empty), Spots Used, TKL flag
**Important Limitation:** NOT dynamic — does not auto-update when routes change in TMW
**Purpose in Practice:** Starting point for LTL dispatch to understand geographic spread and
identify which trips to look at in TMW system
**Automation Target:** Medium — driver/route from TMW, spots used partially from TMW,
empty status requires route completion data
**Dashboard Behavior Required:** Must run on batch timer and update dynamically from TMW data

### 6. Driver Planning Sheet
**Purpose:** Weekly scheduling board for all drivers by day of week
**Frequency:** Weekly reset — NO historical copies preserved
**Structure:** Tabs for each day of week (Sun-Sat), organized by terminal and driver type
**Key Fields:** Start Time, First Stop City, Truck, Trailer, Loaded/Empty, Phone,
Trip Status, Confirm Driver, Current Location, Endorsements, Restrictions
**Auto-populate Candidates:** Driver roster, endorsements, restrictions, phone, truck/trailer
**Human Input Required:** Daily start times, trip status, confirm flags, location notes
**Automation Target:** Partial — roster and static driver data auto-populated,
daily operational status still requires dispatcher input

---

## Dashboard Architecture Requirement
All dashboards must run on a batch timer and update dynamically from TMW data.
Static snapshots are not acceptable — the operational value is in real-time or near-real-time data.
Looker Studio is the target display layer for read-only dashboards.
Google Sheets retained only for collaborative human-input fields.

---

## Data Sources — Complete Map

### 1. TMW (TMWGATE)
Primary operational data source.
- Orders, loads, assignments, trip status
- Driver master records
- Skid counts, origins, destinations
- Event types (XDL, XDU, LLD, LUL etc.)
- Truck and trailer assignments
- Company/customer codes

### 2. Geotab API
Vehicle and driver location data.
- Already integrated by project owner via existing Apps Script automations
- Used in driver planning sheet right-column automations today
- Will be integrated properly in the professional rebuild
- Provides: real-time vehicle location, driver ETA data
- Scope: out of scope for Phase 1 (Load Sheet, OTR boards)
- Scope: required for Phase 2 (Driver Planning Sheet dynamic rebuild)

### 3. Human Input Layer
Operational judgment that lives in no system.
- OTR board planning notes and routing decisions
- Confirm driver flags
- Exception notes and special instructions
- Substitute driver assignments
- Anything dispatchers know that TMW doesn't capture

### DOMO Integration Context
- 24-48 hour lag — reference map only, not pipeline
- 6+ joined datasets built and validated by project owner
- Use DOMO to identify table relationships and join keys
- Validate query results against DOMO (accounting for lag)
- DOMO is not being replaced

---

## ASR / Eleos Reference Architecture
- Built custom API over TMW SQL using stored procedures
- Created dedicated SQL users (WEB_ELEOS, ELEOS, ADMIN_ELEOS)
- TMWGATE_Test originally set up by ASR for their integration testing
- This project follows same pattern at read-only scale

## SystemsLink — Not Used
- ~$30,000 cost, not purchased, not needed
- Reject any suggestion to use SystemsLink

## Net Terms Project — Separate
- Do not mix with dashboard project
- SettlementPaymentTerms, BillingOutputTerm, cmp_Payment_terms reserved for that project

---

## Shuttle Sheet — Detailed Technical Specification

### Business Purpose
Dedicated shuttle routes between cross-dock terminals move LTL freight between locations
for transloading onto terminal-specific trucks. The shuttle sheet is the top section of
each terminal's daily load sheet and is built from early morning until 4-5pm. The load
sheet is then built starting at 4-5pm, incorporating the shuttle data at the top.

### Dedicated Shuttle Routes
1. **GR ↔ Clare Shuttle**
   - GR terminal: Byron Center, MI
   - Dedicated GR driver runs evenings, roughly same time nightly
   - Direction shown in Chad Collins trip folder: GR (pickup) → Clare (delivery)
   - Also runs return direction

2. **Clare ↔ Taylor Shuttle**
   - Starts in Clare, round trip to Taylor cross-dock and back
   - Shuttles LTL orders Clare → Taylor for delivery
   - Picks up freight in Taylor to drop in Clare for transloading onto Clare trucks

### Important Edge Cases
- Sometimes a shuttle leg is a full TKL (not just LTL)
- Multiple shuttle drivers possible beyond just dedicated drivers
- Could be a TKL outbound and LTL return, or any combination
- All variations get marked on the shuttle and load sheets

### How Shuttle Orders Appear on Load Sheets
- Same order number appears on TWO terminal load sheets simultaneously
- On origin terminal sheet: order listed in shuttle section at top, driver = dedicated shuttle driver
- On destination terminal sheet: order listed with "Shuttle" in the driver column
- Example: Order 1070980 appears on GR 410 sheet (top/shuttle section) AND Clare 410 sheet (marked "Shuttle")

### TMW Identification — CONFIRMED from Trip Folder Analysis

⚠️ IMPORTANT: XDL/XDU event types alone are NOT sufficient to identify shuttle orders.
XDL/XDU = any cross-dock anywhere in the network. Overfiltering on this alone will
return non-shuttle cross-docks. Shuttle identification requires ALL three signals combined.

Shuttle orders are identified by the INTERSECTION of all three:

**Signal 1 — Internal Company Codes**
- Company: NORTHERN LOGISTICS - GR
- Company: NORTHERN LOGISTICS - CLARE
- Company: NORTHERN LOGISTICS - TAYLOR (verify)
- These are internal company codes, NOT customer companies
- Regular load sheet orders use external customer company names

**Signal 2 — Double Cross-Dock Event Types**
- XDL = Cross Dock Load (pickup at origin terminal)
- XDU = Cross Dock Unload (delivery at destination terminal)
- Shuttle orders have BOTH XDL and XDU legs in the same trip/movement
- Confirmed from Movement 2003141 (Chad Collins, 4/10/26):
  - Multiple XDL events at Byron Center MI (GR terminal)
  - Multiple XDU events at Clare MI (Clare terminal)

**Other Event Codes Seen**
- LLD = Live Load Delivery
- LUL = Live Unload Load
- These appear on non-shuttle freight in the same trip

### SQL View Strategy for Shuttle Sheet
Filter orders where ALL of the following are true:
1. Company code matches internal Northern Logistics company codes
   (NORTHERN LOGISTICS - GR, NORTHERN LOGISTICS - CLARE, NORTHERN LOGISTICS - TAYLOR)
   AND
2. Terminal city pair matches a known shuttle lane:
   - Byron Center MI ↔ Clare MI (GR shuttle)
   - Clare MI ↔ Taylor MI (Clare-Taylor shuttle)
   AND
3. Freight events contain XDL AND XDU leg types within the same movement

Driver ID as additional stratification layer:
- Primary: filter by dedicated driver IDs (COLCH = Chad Collins for GR shuttle)
- Secondary: when substitute driver covers the route, signals 1+2 still capture the order
- Driver ID confirms dedicated shuttle vs one-off cross-dock on the same lane

This produces the shuttle order list. Join to driver/tractor for the dedicated shuttle driver assignment.

The shuttle view must show:
- Order #
- Origin terminal
- Destination terminal
- Skid count
- Assigned shuttle driver
- Trailer #
- Movement/Trip #
- ETA / departure time

### Relationship to Load Sheet
- Shuttle orders that complete at a terminal FEED INTO that terminal's load sheet
- The bottom of the load sheet shows orders arriving via shuttle from another terminal
- These become available for the terminal's local drivers to deliver onward
- This creates a two-tier structure: shuttle section (top) + terminal delivery section (bottom)

### Document Save Behavior
- Shuttle sheets: Historical copies preserved (same as load sheets)
- Both are remade daily

### Driver Planning Sheet — Dependency on Geotab
The right-column automations in the current driver planning sheet are Apps Script
automations tied to the Geotab API (vehicle tracking). These are out of scope for
Phase 1 but will be rebuilt properly in the driver planning sheet phase using the
same Geotab API access the project owner already has established.
Do not attempt to replicate these automations until the TMW data layer is proven.

### Confirmed Order Cross-Reference (4/10 data)
Orders appearing on BOTH GR 410 and Clare 410 as shuttle:
- 1070980 — Kastalon → Midwest Advanced (3 skids)
- 1071148 — Manor Tool → Robinson Curtis (4 skids)
- 1071175 — Sun Down Sheet Metal → Wilco MFG (2 skids)
- 1071108 — Freedom Finishing → Grant Industries (1 skid)
- 1071168 — Shellcast → East Jordan Ironworks (1 skid)
- 1071102 — Controlled Plating → Rite Hite (7 skids)
