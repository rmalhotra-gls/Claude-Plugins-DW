---

name: sql-deployment-standards
description: "Standards for RUN, ROLLBACK, and MONITORING deployment scripts with environment routing, cross-database references, and validation patterns for GLS Auto Snowflake. Trigger when writing, reviewing, or generating deployment scripts, or when user asks about RUN/ROLLBACK/MONITORING file structure, environment routing, or pre-built variable patterns."

---

## Environment Routing

### Database Variable Initialization

All three files (RUN, ROLLBACK, MONITORING) must begin with the same environment routing block:

```sql
SET DB_ODIN = CASE
    WHEN CURRENT_ROLE() IN ('SYSADMIN', 'DW_ENGINEER_PROD') THEN 'ODIN'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_TEST' THEN 'HEL'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_UAT' THEN 'ODIN_UAT'
END;

SET DB_HEIMDALL = CASE
    WHEN CURRENT_ROLE() IN ('SYSADMIN', 'DW_ENGINEER_PROD') THEN 'HEIMDALL'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_TEST' THEN 'MIDGARD'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_UAT' THEN 'HEIMDALL_UAT'
END;

SET DB_MJOLNIR = CASE
    WHEN CURRENT_ROLE() IN ('SYSADMIN', 'DW_ENGINEER_PROD') THEN 'MJOLNIR'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_TEST' THEN 'STORMBREAKER'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_UAT' THEN 'MJOLNIR_UAT'
END;

SET DB_VALHALLA = CASE
    WHEN CURRENT_ROLE() IN ('SYSADMIN', 'DW_ENGINEER_PROD') THEN 'VALHALLA'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_TEST' THEN 'VIKING'
    WHEN CURRENT_ROLE() = 'DW_ENGINEER_UAT' THEN 'VALHALLA_UAT'
END;
```

**Purpose:** This block ensures every script runs against the correct database for the current environment. A SYSADMIN or DW_ENGINEER_PROD role will hit production (ODIN, HEIMDALL, MJOLNIR); a TEST role will hit lower environments (HEL, MIDGARD, STORMBREAKER); UAT stays in UAT isolation (ODIN_UAT, HEIMDALL_UAT, MJOLNIR_UAT).

### Using Variables in Scripts

Refer to databases using `IDENTIFIER()` with the variables:

```sql
USE DATABASE IDENTIFIER($DB_HEIMDALL);
```

For inline schema references:

```sql
-- Pre-build the reference, then use it
SET V_O_INFO_COLS = $DB_ODIN || '.INFORMATION_SCHEMA.COLUMNS';
SELECT * FROM IDENTIFIER($V_O_INFO_COLS);
```

---

## Stored Procedure Cross-Database References

### Inside Stored Procedure Logic

When a stored procedure needs to reference a different database dynamically, map the calling database to the correct target database:

```sql
CREATE OR REPLACE PROCEDURE SP_PROCESS_DATA()
LANGUAGE SQL
EXECUTE AS CALLER
AS
DECLARE
    CURRENTDATABASE VARCHAR;
    DBODIN VARCHAR;
BEGIN
    -- Determine which database we're running in
    SET CURRENTDATABASE = CURRENT_DATABASE();

    -- Map to the correct ODIN variant
    IF (CURRENTDATABASE = 'HEIMDALL') THEN
        SET DBODIN = 'ODIN';
    ELSEIF (CURRENTDATABASE = 'HEIMDALL_UAT') THEN
        SET DBODIN = 'ODIN_UAT';
    ELSEIF (CURRENTDATABASE = 'MIDGARD') THEN
        SET DBODIN = 'HEL';
    END IF;

    -- Now reference DBODIN in queries
    EXECUTE IMMEDIATE 'SELECT * FROM ' || DBODIN || '.INFORMATION_SCHEMA.TABLES';
END;
```

**Why:** Cross-database references in Snowflake cannot always use variables in view definitions. This pattern allows stored procedures to remain environment-agnostic while maintaining strict routing.

---

## Ad-Hoc Scripts: Pre-Built Table References

For ad-hoc scripts, backfill updates, and monitoring queries, pre-build fully qualified table references using variables:

### Variable Declaration Block

```sql
SET V_H_JSON_HIST        = $DB_HEIMDALL || '.STAGEJSON.DEFIDEALERHISTORY';
SET V_O_INFO_COLS        = $DB_ODIN     || '.INFORMATION_SCHEMA.COLUMNS';
SET V_H_INFO_COLS        = $DB_HEIMDALL || '.INFORMATION_SCHEMA.COLUMNS';
SET V_O_DW_TABLES        = $DB_ODIN     || '.INFORMATION_SCHEMA.TABLES';
SET V_H_ETL_PROC_CONFIG  = $DB_HEIMDALL || '.ETL.PROCESSCONFIG';
```

### Using Pre-Built References in Queries

```sql
SELECT
    'CHECK 01C: DEFIDEALEREXTRACT - CUSTOMACTIVEFORSALIENT exists' AS CHECK_NAME,
    CASE
        WHEN COUNT(*) > 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    COUNT(*) AS DETAIL_COUNT
FROM
    IDENTIFIER($V_H_INFO_COLS)
WHERE
    TABLE_SCHEMA = 'STAGEJSON'
    AND TABLE_NAME = 'DEFIDEALERHISTORY'
    AND COLUMN_NAME = 'CUSTOMACTIVEFORSALIENT';
```

**Benefit:** Once pre-built, references are consistent across the script, readable, and reusable in multiple queries.

---

## Cross-Database Views

### Dynamic Database Routing via Anonymous Block

When a view must reference a different database and you don't want to maintain separate environment-specific copies, use an anonymous EXECUTE IMMEDIATE block to build the CREATE VIEW DDL with the resolved database variable:

```sql
-- Set environment routing first (full 4-database block above)

USE DATABASE IDENTIFIER($DB_MJOLNIR);

EXECUTE IMMEDIATE $$
DECLARE
    V_DB_ODIN VARCHAR;
BEGIN
    SELECT $DB_ODIN INTO :V_DB_ODIN;

    EXECUTE IMMEDIATE
        'CREATE OR REPLACE VIEW INTELLIGENCE.VW_SALESAGENTAGEDTITLES AS
         SELECT
             A.AGENTID,
             A.TITLE,
             A.AGEBAND,
             D.DEALERNAME
         FROM ' || :V_DB_ODIN || '.DW.DIM_SALESAGENT A
             INNER JOIN ' || :V_DB_ODIN || '.DW.DIM_DEALER D
                 ON A.DEALERID = D.DEALERID
         WHERE
             A.ISACTIVE = TRUE';
END;
$$;
```

**Why this works:** EXECUTE IMMEDIATE resolves the database variable at runtime and bakes the literal database name into the compiled view definition. The view itself ends up with a hardcoded reference, but the *script* that creates it is environment-agnostic — same script runs in TEST, UAT, and PROD without modification.

---

## File Structure

### RUN.sql

**Purpose:** Create or modify target objects; insert seed data; execute transformations. Keep note of managing ad-hoc inserts against RUN File accidental re-runs.

**Structure:**

```sql
-- 1. Environment routing
SET DB_ODIN = CASE ...
SET DB_HEIMDALL = CASE ...
SET DB_MJOLNIR = CASE ...

-- 2. Set database context
USE DATABASE IDENTIFIER($DB_HEIMDALL);

-- 3. Pre-build variable references (if ad-hoc script)
SET V_H_JSON_HIST = ...
SET V_O_DW_TABLES = ...

-- 4. Object creation / modification
CREATE TABLE IF NOT EXISTS STAGEJSON.DEFIDEALERHISTORY (...)
CREATE OR REPLACE VIEW POWERBI.DEALERVIEW AS ...
CREATE OR REPLACE PROCEDURE SP_LOADCONTACTS(...) AS ...

-- 5. Data operations (inserts, updates, merges)
INSERT INTO HEIMDALL.STAGEJSON.DEFIDEALERHISTORY (...) SELECT ...
```

### ROLLBACK.sql

**Purpose:** Safely revert all changes introduced by RUN.sql.

**Structure:**

```sql
-- 1. Environment routing (same as RUN.sql)
SET DB_ODIN = CASE ...
SET DB_HEIMDALL = CASE ...
SET DB_MJOLNIR = CASE ...

-- 2. Set database context
USE DATABASE IDENTIFIER($DB_HEIMDALL);

-- 3. Reverse all operations in reverse order
-- Drop new views before tables they depend on
DROP VIEW IF EXISTS POWERBI.DEALERVIEW;

-- Drop new procedures
DROP PROCEDURE IF EXISTS SP_LOADCONTACTS(...);

-- Drop new tables or truncate if table existed pre-deploy
TRUNCATE TABLE IF EXISTS STAGEJSON.DEFIDEALERHISTORY;

-- Restore previous data if applicable
-- (usually handled by pre-deploy backup or incremental watermark reset)
```

**Best Practice:** ROLLBACK should never require manual intervention. If data was inserted, truncate. If a view was created, drop it. If a procedure was modified, recompile the old version. Test ROLLBACK in lower environments before production deployment.

### MONITORING.sql

**Purpose:** Validate that RUN.sql executed correctly. Returns standardized schema for easy interpretation.

**Structure:**

```sql
-- 1. Environment routing (same as RUN.sql)
SET DB_ODIN = CASE ...
SET DB_HEIMDALL = CASE ...
SET DB_MJOLNIR = CASE ...

-- 2. Set database context
USE DATABASE IDENTIFIER($DB_HEIMDALL);

-- 3. Pre-build variable references
SET V_H_JSON_HIST = $DB_HEIMDALL || '.STAGEJSON.DEFIDEALERHISTORY';
SET V_H_INFO_COLS = $DB_HEIMDALL || '.INFORMATION_SCHEMA.COLUMNS';

-- 4. Run all checks as UNION ALL (single SELECT)
SELECT
    'PRE_01_TARGET_TABLE_EXISTS' AS CHECKNAME,
    CASE
        WHEN COUNT(*) = 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'Target table does not exist pre-deploy' AS EXPECTED,
    COUNT(*) || ' tables found' AS ACTUAL
FROM
    IDENTIFIER($V_H_INFO_COLS)
WHERE
    TABLE_SCHEMA = 'STAGEJSON'
    AND TABLE_NAME = 'DEFIDEALERHISTORY'

UNION ALL

SELECT
    'POST_01_TABLE_COLUMN_COUNT' AS CHECKNAME,
    CASE
        WHEN ACTUAL_COUNT = 8 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    '8 columns in target table' AS EXPECTED,
    ACTUAL_COUNT || ' columns found'
        || CASE WHEN ACTUAL_COUNT != 8
            THEN ' | present: ' || ACTUAL_COLUMNS
            ELSE '' END AS ACTUAL
FROM
    (
        SELECT
            COUNT(*) AS ACTUAL_COUNT,
            LISTAGG(COLUMN_NAME, ', ') WITHIN GROUP (ORDER BY ORDINAL_POSITION) AS ACTUAL_COLUMNS
        FROM
            IDENTIFIER($V_H_INFO_COLS)
        WHERE
            TABLE_SCHEMA = 'STAGEJSON'
            AND TABLE_NAME = 'DEFIDEALERHISTORY'
    )

UNION ALL

SELECT
    'POST_02_TABLE_HAS_DATA' AS CHECKNAME,
    CASE
        WHEN ROW_COUNT > 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'Target table has rows after RUN' AS EXPECTED,
    ROW_COUNT || ' rows loaded' AS ACTUAL
FROM
    (
        SELECT COUNT(*) AS ROW_COUNT
        FROM IDENTIFIER($V_H_JSON_HIST)
    )

UNION ALL

SELECT
    'POST_03_NO_DUPLICATES' AS CHECKNAME,
    CASE
        WHEN DUP_COUNT = 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'No duplicate DEALERID values in active set' AS EXPECTED,
    DUP_COUNT || ' duplicates: ' || DUP_LIST AS ACTUAL
FROM
    (
        SELECT
            COUNT(*) AS DUP_COUNT,
            LISTAGG(DEALERID, ', ') WITHIN GROUP (ORDER BY DEALERID) AS DUP_LIST
        FROM
            (
                SELECT DEALERID
                FROM IDENTIFIER($V_H_JSON_HIST)
                WHERE ISACTIVE = TRUE
                GROUP BY DEALERID
                HAVING COUNT(*) > 1
            )
    );
```

---

## MONITORING.sql Output Schema

Every check must return exactly 4 columns:

| Column | Purpose |
|---|---|
| `CHECKNAME` | Short identifier. Pattern: `PRE_##_LABEL` or `POST_##_LABEL`. |
| `RESULT` | `PASS` or `FAIL`. Never just one word in the ELSE — always include diagnostic detail in ACTUAL. |
| `EXPECTED` | What the check is looking for. Static text is fine. |
| `ACTUAL` | **What was actually found.** This is the diagnostic payload. On a FAIL, must contain enough detail to diagnose without follow-up queries. |



### Check Categories

**Pre-deploy checks (PRE_):**
- Target objects don't exist yet (or do exist if modifying)
- Config rows don't exist yet
- Source objects exist and have data

**Post-deploy structural checks (POST_):**
- Objects created with correct column count and names
- Key columns are in correct position with correct constraints (NOT NULL, AUTOINCREMENT)
- Views return expected column set
- Stored procedures exist
- Config rows exist with correct settings

**Post-deploy data checks (POST_):**
- Target table has rows after execution
- No duplicate active/current rows per natural key
- All values in constrained columns within expected domain
- Foreign key integrity holds
- Exclusion filters work correctly
- Source objects still function after modification
