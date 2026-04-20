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

### Source Control Rules
- main branch = confirmed working code only.
- dev branch = all active development and testing.
- Never instruct user to push directly to main without confirming the change works first.
- Every meaningful change gets committed before moving to the next step.

### API Rules
- x-api-key header validation on every endpoint. No exceptions.
- Never expose raw SQL error messages in API responses.
- Return clean JSON errors only.
- One endpoint per dataset. Keep it simple.

### Build Rules
- One step at a time. Confirm each step works before writing the next.
- Never write more than one build step ahead of confirmed results.
- If a query takes more than 3 seconds in SSMS it needs to be rewritten before use.

---

## System Architecture

```
Layer 1 — External Client
  Looker Studio / Google Sheets
  - HTTP calls to API only
  - Sends x-api-key header
  - NEVER connects directly to SQL Server
  - NEVER contains business logic

Layer 2 — API (Node.js Express)
  - Validates API key on every request
  - Calls SQL layer via views or stored procs only
  - Returns JSON
  - Handles errors without exposing SQL internals
  - MUST NOT query raw tables directly
  - MUST NOT contain inline complex SQL

Layer 3 — SQL Access Layer
  - ALL database access via:
      Views → simple read-only SELECT datasets
      Stored Procedures → filtered or parameterized queries
  - Naming: vw_LoadSheet / vw_OTRBoard / vw_DriverPlanning
  - Naming: sp_GetLoadSheetByTerminal / sp_GetActiveOTR
  - This layer is the safety boundary

Layer 4 — Database
  Development: TMWGATE_Test (frozen 2024 snapshot — structural testing only)
  Production:  TMWGATE (read-only SELECT via tmw_api_user)
```

---

## SQL User Setup

```sql
-- Run in TMWGATE_Test first, then repeat in TMWGATE for production
CREATE LOGIN tmw_api_user WITH PASSWORD = '[value from .env — never hardcode]';

USE TMWGATE_Test; -- switch to TMWGATE for production step
CREATE USER tmw_api_user FOR LOGIN tmw_api_user;

-- Grant only what is needed — never db_datareader (too broad)
GRANT SELECT ON dbo.vw_LoadSheet TO tmw_api_user;
GRANT SELECT ON dbo.vw_OTRBoard TO tmw_api_user;
GRANT SELECT ON dbo.vw_DriverPlanning TO tmw_api_user;
GRANT EXECUTE ON dbo.sp_GetLoadSheetByTerminal TO tmw_api_user;
```

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

```javascript
try {
  const result = await pool.request().query('SELECT * FROM vw_LoadSheet');
  res.json(result.recordset);
} catch (err) {
  console.error('Internal query error:', err); // log internally only
  res.status(500).json({ error: 'Data retrieval failed' }); // clean external response
}
```

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
