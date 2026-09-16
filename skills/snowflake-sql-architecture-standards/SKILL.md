---
name: snowflake-sql-architecture-standards
description: GLS Auto Snowflake SQL architecture standards reference and enforcer. Use this skill whenever the user is writing, reviewing, or asking about SQL code, stored procedures, views, table design, naming conventions, schema structure, incremental loads, IT fields, DQ framework, comments, or reporting objects for the GLS Auto data warehouse. Always trigger when the user asks "is this correct", "does this follow standards", "what should I name this", "how should I structure this", or pastes any Snowflake SQL for feedback or generation. Also trigger when the user asks about sproc patterns, view naming, hist table design, error logging, column comments, or PowerBI schema objects. These are FORWARD-LOOKING standards — existing objects in ODIN.DW are legacy and may not comply.
---

# GLS Auto — Snowflake SQL Architecture Standards

> **Scope:** These are forward-looking standards for all NEW objects built in the GLS Auto Snowflake environment. Existing objects in `ODIN.DW` are legacy and may not comply — do not use them as reference for how new objects should be built. When touching existing objects, migrate them toward these standards.

---

## 1. Environment & Schema Structure

### Primary Database
All DW work lives in the **`ODIN`** database.

### Schema Layer Structure

Two valid pipeline configurations depending on use case:

**4-Layer (Full Pipeline):**
```
SRC / STG<domain> → STG → DW → POWERBI
```

**3-Layer (Simplified Pipeline):**
```
SRC / STG<domain> → STG → DW
```

| Schema | Purpose |
|---|---|
| `SRC` | All inbound vendor feeds and flat files — raw, untransformed |
| `STG` / `STG<domain>` | Staging schemas (e.g., `STGSALES`, `STGCCI`, `STGREFI`) — cleansing, joins, intermediate transforms per domain |
| `DW` | Main data warehouse — all FACT, DIM, LKP tables and their load sprocs |
| `POWERBI` | Reporting views (`VW_`) exposed to Power BI |
| `ETL` | Monitoring, logging, and execution tracking objects |
| `TEMP` | Temporary/backup tables during story work only (see naming rules) |
| `RPT` | Additional reporting objects |
| `DWLIMITED` | Objects restricted to DW-access only |

> **Note:** Schema-level abbreviations (`STG`, `DW`, `SRC`) are inherited legacy conventions and are the only place abbreviations are permitted. All object names and column names within those schemas must use full words. See Section 3.

---

## 2. Mandatory Comments — Every Object, Every Column

**This is non-negotiable.** Every object and every column must have a comment. No exceptions.

### Why
Comments are the first line of defense when something breaks. They answer "what is this, where does it come from, and who owns it?" without needing to track someone down.

### Table / View Comments
Set at DDL time using `COMMENT = '...'`:

```sql
CREATE OR REPLACE TABLE ODIN.DW.FACTLOANAPPLICATION
COMMENT = 'One row per loan application submitted through the GLS origination pipeline. Grain: one record per ApplicationID. Source: ODIN.STG via LOAD_FACTLOANAPPLICATION.'
(
	...
);

-- Or post-creation:
ALTER TABLE ODIN.DW.FACTLOANAPPLICATION
SET COMMENT = 'One row per loan application submitted through the GLS origination pipeline.';
```

### Column Comments
Inline in the column definition:

```sql
CREATE OR REPLACE TABLE ODIN.DW.FACTLOANAPPLICATION
COMMENT = 'One row per loan application. Grain: ApplicationID.'
(
	APPLICATIONID TEXT NOT NULL COMMENT 'Unique application identifier from DeFi LOS. Natural key.',
	DEALID TEXT COMMENT 'Associated deal ID. NULL if application has not been booked.',
	LOANAMOUNT NUMBER(18, 2) COMMENT 'Requested loan amount in USD at time of application.',
	APPLICATIONSTATUS TEXT COMMENT 'Current status code. Joins to ODIN.DW.LKPAPPLICATIONSTATUS.',
	-- ... business columns ...
	IT_INSERTDATE TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP() COMMENT 'Row insert timestamp. Auto-populated on load.',
	IT_UPDATEDATE TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP() COMMENT 'Row last update timestamp. Updated on each MERGE.',
	IT_LOANAPPLICATIONKEY NUMBER COMMENT 'Surrogate key. System-generated. Do not use as join key in reports.'
);
```

### Stored Procedure Comments
Every sproc must have both an object-level `COMMENT` and an inline header block:

```sql
CREATE OR REPLACE PROCEDURE ODIN.DW.LOAD_FACTLOANAPPLICATION()
RETURNS VARCHAR(20)
LANGUAGE SQL
COMMENT = 'Incrementally loads loan application data from ODIN.STG into ODIN.DW.FACTLOANAPPLICATION. Runs daily. Called by LOAD_MASTER.'
AS
$$
-- ============================================================
-- Procedure  : LOAD_FACTLOANAPPLICATION
-- Schema     : ODIN.DW
-- Source     : ODIN.STG.<source_table>
-- ------------------------------------------------------------
-- Change History (keep last 4 — drop oldest on new entry):
-- Date        | User      | Description
-- ------------|-----------|-----------------------------------
-- 2025-01-15  | rmalhotra | Initial creation
-- ============================================================
...
$$;
```

---

## 3. Object Naming Conventions

### Rule 1 — No Abbreviations. Ever.
Object names and column names must use **full, descriptive words**. The name alone must make the purpose of the object immediately obvious to someone unfamiliar with the codebase. If a reasonable person reading the name for the first time would have to guess what it means, the name is not acceptable.

| ❌ Not allowed | ✅ Required |
|---|---|
| `FACTAPPSTAT` | `FACTAPPLICATIONSTATUS` |
| `DIMDLRINFO` | `DIMDEALERINFO` |
| `LKPLNSTS` | `LKPLOANSTATUS` |
| `LOAD_APPLS` | `LOAD_FACTLOANAPPLICATION` |
| `APPID` | `APPLICATIONID` |
| `AMT` | `AMOUNT` |
| `DT` | `DATE` |
| `QTY` | `QUANTITY` |
| `STS` | `STATUS` |
| `ORIG` (as column name) | `ORIGINATION` |
| `DLRID` | `DEALERID` |
| `PYMTAMT` | `PAYMENTAMOUNT` |
| `CREATEDT` | `CREATIONDATE` |

> Schema-level names (`STG`, `DW`, `SRC`) are inherited legacy conventions and are the **only** place abbreviations are permitted. Everywhere else — full words.

### Rule 2 — Underscore Rules

**FACT, DIM, LKP** — the prefix is fused directly to the name. No underscore.

**LOAD_, UPDATE_, VW_, IT_** — these keep their underscores.

**_HIST, _DATA<story#>** — these suffixes keep their underscores.

| Object type | Pattern | Example |
|---|---|---|
| Fact table | `FACT<FullName>` | `FACTLOANAPPLICATION` |
| Dimension table | `DIM<FullName>` | `DIMDEALERINFO` |
| Lookup table | `LKP<FullName>` | `LKPLOANSTATUS` |
| Hist table | `<FullTableName>_HIST` | `FACTLOANAPPLICATION_HIST` |
| Temp / backup | `<FullTableName>_DATA<Story#>` | `FACTLOANAPPLICATION_DATA1042` |
| Load sproc | `LOAD_<FullTableName>` | `LOAD_FACTLOANAPPLICATION` |
| Update sproc | `UPDATE_<FullTableName>` | `UPDATE_FACTLOANAPPLICATION` |
| Reporting view | `VW_<FullReportName>_<FullTabName>` | `VW_DealerPerformance_ApplicationSummary` |
| IT insert timestamp | `IT_INSERTDATE` | — |
| IT update timestamp | `IT_UPDATEDATE` | — |
| IT surrogate key | `IT_<tablename>KEY` | `IT_LOANAPPLICATIONKEY` |
| Business column | No underscores, all caps, full words | `LOANAMOUNT`, `APPLICATIONSTATUS` |

### Rule 3 — All Caps
All object names and column names are uppercase.

---

### Columns — IT (System/Audit) Fields
IT fields go **at the end of every table definition**. No exceptions.

| Field | Pattern | Data Type | Constraint |
|---|---|---|---|
| Insert timestamp | `IT_INSERTDATE` | `TIMESTAMP_NTZ` | `NOT NULL DEFAULT CURRENT_TIMESTAMP()` |
| Update timestamp | `IT_UPDATEDATE` | `TIMESTAMP_NTZ` | `NOT NULL DEFAULT CURRENT_TIMESTAMP()` |
| Surrogate key | `IT_<tablename>KEY` | `NUMBER` | — |

> **IT key rule:** Strip `FACT`, `DIM`, `LKP` from the table name — no abbreviation in the remainder.
> `FACTLOANAPPLICATION` → `IT_LOANAPPLICATIONKEY`
> `DIMDEALERINFO` → `IT_DEALERINFOKEY`
> `LKPLOANSTATUS` → `IT_LOANSTATUSKEY`

---

## 4. Stored Procedure Standards

### Return Type
```sql
RETURNS VARCHAR(20)
```

### Full Sproc Template

```sql
CREATE OR REPLACE PROCEDURE ODIN.DW.LOAD_<FULLTABLENAME>()
RETURNS VARCHAR(20)
LANGUAGE SQL
COMMENT = '<One sentence: what this loads, where it comes from, what calls it.>'
AS
$$
-- ============================================================
-- Procedure  : LOAD_<FULLTABLENAME>
-- Schema     : ODIN.DW
-- Source     : ODIN.<STG_SCHEMA>.<FULL_SOURCE_TABLE_NAME>
-- ------------------------------------------------------------
-- Change History (keep last 4 — drop oldest on new entry):
-- Date        | User      | Description
-- ------------|-----------|-----------------------------------
-- YYYY-MM-DD  | username  | Initial creation
-- ============================================================

DECLARE
	V_RESULT VARCHAR(20);

BEGIN
	-- [your logic here]

	V_RESULT := 'SUCCESS';
	RETURN V_RESULT;

EXCEPTION
	WHEN OTHER THEN
		RETURN 'ERROR: ' || SQLERRM;
END;
$$;
```

### Rules
- Change history: **Date, User, Description** — max 4 entries, drop oldest when adding
- `Source` = full schema.table path where source data originates — no abbreviations
- No `SELECT *` anywhere — always explicit column lists
- IT fields must be in all INSERT/MERGE target column lists
- DQ row-count check before committing (see Section 7)

---

## 5. Table DDL Template

```sql
CREATE OR REPLACE TABLE ODIN.DW.FACT<FULLTABLENAME>
COMMENT = 'One row per <X> per <Y>. Source: ODIN.<STG>.<FULLTABLENAME>. Loaded by LOAD_<FULLTABLENAME>.'
(
	-- Business columns
	<FULLCOLUMNNAME1> <DATATYPE> NOT NULL COMMENT '<What this is, where it comes from.>',
	<FULLCOLUMNNAME2> <DATATYPE> COMMENT '<What this is. NULL when X.>',

	-- IT fields — always last
	IT_INSERTDATE TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP() COMMENT 'Row insert timestamp. Auto-populated.',
	IT_UPDATEDATE TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP() COMMENT 'Row last update timestamp. Updated on MERGE.',
	IT_<FULLTABLENAME>KEY NUMBER COMMENT 'Surrogate key. System-generated.'
);
```

---

## 6. Incremental Load Pattern (HIST Tables)

When a table has an incremental load, a corresponding `_HIST` table captures **daily deltas only** — not full history replays.

**Rationale:** Historical data doesn't change. The `_HIST` table answers "what changed yesterday?" It is append-only and must never be truncated or reloaded.

### Pattern

```sql
-- Daily incremental delta — appended after main table load
INSERT INTO ODIN.DW.FACTLOANAPPLICATION_HIST
SELECT
	<ALL COLUMNS INCLUDING IT FIELDS>
FROM
	ODIN.DW.FACTLOANAPPLICATION
WHERE
	IT_INSERTDATE >= DATEADD(DAY, -1, CURRENT_DATE())
	AND IT_INSERTDATE < CURRENT_DATE();
```

### Rules
- `_HIST` table mirrors the parent table's column structure exactly — same columns, same comments
- IT fields always carried over
- Daily cadence
- Append only — never truncate or reload

---

## 7. DQ Framework & Error Logging

Snowflake native error logging is enabled on tables. A formal framework is in progress — apply these patterns on all new objects.

### Standards
- Error tables must be **queryable and actionable** — not passive logs
- DQ row-count checks must be embedded in `LOAD_` sprocs **before** committing data
- On failure, sproc must return a clearly parseable error string so Prefect can catch and alert

### DQ Check Pattern

```sql
-- Inside LOAD_ sproc, before INSERT/MERGE:
LET V_SOURCEROWCOUNT INT := (
	SELECT
		COUNT(*)
	FROM
		ODIN.STG.LOANAPPLICATION
	WHERE
		LOADDATE >= CURRENT_DATE()
);

IF (V_SOURCEROWCOUNT = 0) THEN
	RETURN 'ERROR: No source rows found for ' || CURRENT_DATE()::VARCHAR;
END IF;
```

---

## 8. Reporting Standards (ODIN.POWERBI)

- Views live in `ODIN.POWERBI`, named `VW_<FullReportName>_<FullTabName>`
- Reporting sprocs live in `ODIN.DW`
- Tab names = Power BI dataset names — spaces allowed in both
- All views must have a `COMMENT` on the object and on every column
- New reporting objects must follow `VW_` naming — migrate existing non-compliant objects when touched

---

## 9. Business vs. Engineering Naming Balance

- Business stakeholders should recognize column names without a data dictionary
- Engineering standards (no abbreviations, no underscores in business column names, IT fields last, prefixes) always apply
- When in conflict: **engineering standards win — document the business meaning in the column comment**

---

## 10. Kimball Dimensional Modeling — FACT vs DIM vs LKP

### FACT Tables — `FACT<Name>`

**Use when:** Data represents a **business event or transaction** — something that happened and can be measured.

**Ask:** *"Is this something that happened, and can I count or sum it?"* → Yes → FACT

| Kimball Fact Type | When to use | Example |
|---|---|---|
| **Transaction** | One row per discrete event | Application submitted, payment received |
| **Periodic Snapshot** | One row per entity per period | Daily loan balance per account |
| **Accumulating Snapshot** | One row per entity lifetime; milestone columns update | Loan lifecycle: applied → approved → funded → closed |

**Grain rule — declare before building:**
> *"One row in this table represents one [event] per [entity]."*
> If you can't state this clearly, the design is not ready.

---

### DIM Tables — `DIM<Name>`

**Use when:** Data represents a **business entity** — a person, place, product, or concept that gives context to facts.

**Ask:** *"Is this a 'who', 'what', 'where', or 'which' that describes something that happened?"* → Yes → DIM

| SCD Type | Behavior | GLS Default |
|---|---|---|
| **Type 1** | Overwrite — no history | **Default.** Use for corrections and non-material changes. |
| **Type 2** | New row per change — full history | Use `_HIST` pattern when business needs point-in-time history |
| **Type 3** | Add column — current + prior only | Rarely used |

---

### LKP Tables — `LKP<Name>`

**Use when:** Data is a **small, stable code/label reference list** — not a full entity, not a transaction.

**Ask:** *"Is this just a list of codes decoding into labels?"* → Yes → LKP

| Scenario | Use |
|---|---|
| Codes + labels only, no business attributes | `LKP` |
| Has attributes, joins to multiple tables, or needs SCD | `DIM` |
| < 500 rows, changes rarely | `LKP` |
| > 1,000 rows and growing | Likely `DIM` |

---

### Decision Tree

```
Is this data about something that HAPPENED (event / transaction)?
│
├── YES → FACT<FullName>
│         └── Declare grain first. Then:
│               ├── One event per row           → Transaction Fact
│               ├── Status snapshot per period  → Periodic Snapshot Fact
│               └── Lifecycle with milestones   → Accumulating Snapshot Fact
│
└── NO → Is this a business ENTITY with descriptive attributes?
          │
          ├── YES → DIM<FullName>
          │         └── Does history of changes matter to the business?
          │               ├── No  → SCD Type 1 (overwrite)
          │               └── Yes → SCD Type 2 (_HIST pattern)
          │
          └── NO → Is this a small, stable CODE / LABEL list?
                    │
                    ├── YES → LKP<FullName>
                    └── NO  → Reconsider — likely needs to split into FACT + DIM
```

### Conformed Dimensions
If a dimension is used by more than one fact table, it must be conformed — same `IT_` key, same grain, same column definitions across all referencing fact tables.

---

## 11. Quick Reference — Naming Cheat Sheet

| Object | Pattern | Example |
|---|---|---|
| Fact table | `ODIN.DW.FACT<FullName>` | `ODIN.DW.FACTLOANAPPLICATION` |
| Dim table | `ODIN.DW.DIM<FullName>` | `ODIN.DW.DIMDEALERINFO` |
| Lookup table | `ODIN.DW.LKP<FullName>` | `ODIN.DW.LKPLOANSTATUS` |
| Hist table | `ODIN.DW.<FullTable>_HIST` | `ODIN.DW.FACTLOANAPPLICATION_HIST` |
| Temp / backup | `ODIN.TEMP.<FullTable>_DATA<Story#>` | `ODIN.TEMP.FACTLOANAPPLICATION_DATA2081` |
| Load sproc | `ODIN.DW.LOAD_<FullTableName>` | `ODIN.DW.LOAD_FACTLOANAPPLICATION` |
| Update sproc | `ODIN.DW.UPDATE_<FullTableName>` | `ODIN.DW.UPDATE_FACTLOANAPPLICATION` |
| Reporting view | `ODIN.POWERBI.VW_<FullReport>_<FullTab>` | `ODIN.POWERBI.VW_DealerPerformance_ApplicationSummary` |
| IT insert timestamp | `IT_INSERTDATE` | `TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP()` |
| IT update timestamp | `IT_UPDATEDATE` | `TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP()` |
| IT surrogate key | `IT_<tablename>KEY` | `IT_LOANAPPLICATIONKEY` |
| Business column | Full words, no underscores, all caps | `LOANAMOUNT`, `APPLICATIONSTATUS`, `PAYMENTDATE` |
| Staging schemas | `ODIN.STG`, `ODIN.STG<DOMAIN>` | `ODIN.STGSALES`, `ODIN.STGREFI` |
| Source schema | `ODIN.SRC` | Vendor feeds / flat files |
