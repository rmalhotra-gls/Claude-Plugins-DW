---
name: monitoring-file-generator
description: "Generates MONITORING.sql validation scripts for GLS Auto Snowflake deployments. Trigger on: 'generate monitoring', 'generate validate', 'monitoring script for', 'validation script', 'write the monitoring file', or when a build session completes and MONITORING.sql is a listed deliverable. Can be invoked standalone or as part of the peer review / build workflow."
---

# MONITORING.sql File Generator

Generates standardized MONITORING.sql validation scripts for Snowflake deployments at GLS Auto.

---

## When This Skill Fires

- Engineer says "generate monitoring script for [ticket/objects]"
- Engineer says "write the monitoring file", "generate validate", "validation script"
- Build session completes and MONITORING.sql is a listed deliverable
- Peer reviewer identifies missing MONITORING.sql and engineer asks for one
- Standalone invocation: "monitoring script for DATA-XXXX"

---

## Inputs Required

Before generating, gather:

1. **What objects were created or modified** — tables, views, sprocs, config rows
2. **What data was loaded or changed** — inserts, updates, merges, backfills
3. **What the story requires** — from Jira or locked spec
4. **Live Snowflake state** — query INFORMATION_SCHEMA, GET_DDL, or row counts as needed

If invoked standalone (no prior build context), pull the Jira story first and query Snowflake to understand what exists.

---

## Output Structure — Always This Shape

```sql
-- ============================================================
-- MONITORING.sql — [TICKET-ID]: [Short Description]
-- Generated: [DATE]
-- ============================================================

-- 1. Environment Routing
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

-- 2. Set Database Context
USE DATABASE IDENTIFIER($DB_<PRIMARY>);

-- 3. Pre-Built Variable References
SET V_<ALIAS> = $DB_<X> || '.<SCHEMA>.<TABLE>';
-- ... one per referenced object ...

-- 4. Validation Checks — Single UNION ALL Query
SELECT ... UNION ALL SELECT ... ;
```

---

## Pre-Built Variable References

Every database-qualified table reference used in the checks must be declared as a pre-built variable. No inline IDENTIFIER() concatenation.

**Pattern:**
```sql
SET V_O_DW_TABLE       = $DB_ODIN     || '.DW.FACTLOANAPPLICATION';
SET V_O_INFO_COLS      = $DB_ODIN     || '.INFORMATION_SCHEMA.COLUMNS';
SET V_O_INFO_TABLES    = $DB_ODIN     || '.INFORMATION_SCHEMA.TABLES';
SET V_H_ETL_CONFIG     = $DB_HEIMDALL || '.ETL.PROCESSCONFIG';
```

**Naming convention:** `V_<DB_INITIAL>_<SHORT_LABEL>`
- `V_O_` = ODIN reference
- `V_H_` = HEIMDALL reference
- `V_M_` = MJOLNIR reference
- `V_V_` = VALHALLA reference

---

## Check Output Schema — 4 Columns, No Exceptions

Every check returns exactly these 4 columns:

| Column | Purpose |
|---|---|
| `CHECKNAME` | Short identifier. Pattern: `PRE_##_LABEL` or `POST_##_LABEL`. Uppercase, underscores. |
| `RESULT` | `PASS` or `FAIL`. Nothing else. |
| `EXPECTED` | What the check is looking for. Static text. |
| `ACTUAL` | What was found. On FAIL, must contain enough diagnostic detail to diagnose without follow-up queries. |

---

## Check Categories — What to Include

Generate checks based on what the deployment does. Not every deployment needs every category.

### PRE_ Checks (Pre-Deploy State)

Use when the deployment creates new objects or inserts new data:

| Check | When to Include |
|---|---|
| Target table/view does not exist yet | New table or view being created |
| Config rows do not exist yet | New config inserts |
| Source objects exist and have data | Script reads from source tables |

### POST_ Structural Checks

Always include for any object creation or modification:

| Check | When to Include |
|---|---|
| Object exists after deployment | Any CREATE |
| Column count matches expected | New tables |
| Specific columns exist by name | New or altered tables |
| Key columns have correct constraints (NOT NULL, DEFAULT) | New tables |
| Views return expected column set | New or modified views |
| Stored procedures exist | New or modified sprocs |
| Config rows exist with correct values | Config inserts |

### POST_ Data Checks

Include when the deployment loads or modifies data:

| Check | When to Include |
|---|---|
| Target table has rows after execution | Any data load |
| No duplicate active/current rows per natural key | Tables with active/current flag |
| Values in constrained columns within expected domain | Columns with known valid values |
| Foreign key integrity holds | JOINs to dimension/lookup tables |
| Source objects still function after modification | Modified views or sprocs |
| Row count within expected range | Backfills, bulk loads |
| Backfilled column has no remaining NULLs (or expected NULL count) | Backfill scripts |

---

## Check Pattern Templates

### Object Existence Check
```sql
SELECT
    'POST_01_TABLE_EXISTS' AS CHECKNAME,
    CASE
        WHEN COUNT(*) > 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'Table <SCHEMA>.<TABLE> exists after deployment' AS EXPECTED,
    COUNT(*) || ' tables found' AS ACTUAL
FROM
    IDENTIFIER($V_O_INFO_TABLES)
WHERE
    TABLE_SCHEMA = '<SCHEMA>'
    AND TABLE_NAME = '<TABLE>'
```

### Column Count Check
```sql
SELECT
    'POST_02_COLUMN_COUNT' AS CHECKNAME,
    CASE
        WHEN ACTUAL_COUNT = <N> THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    '<N> columns expected' AS EXPECTED,
    ACTUAL_COUNT || ' columns found'
        || CASE WHEN ACTUAL_COUNT != <N>
            THEN ' | present: ' || ACTUAL_COLUMNS
            ELSE '' END AS ACTUAL
FROM
(
    SELECT
        COUNT(*) AS ACTUAL_COUNT,
        LISTAGG(COLUMN_NAME, ', ') WITHIN GROUP (ORDER BY ORDINAL_POSITION) AS ACTUAL_COLUMNS
    FROM
        IDENTIFIER($V_O_INFO_COLS)
    WHERE
        TABLE_SCHEMA = '<SCHEMA>'
        AND TABLE_NAME = '<TABLE>'
)
```

### Specific Column Exists Check
```sql
SELECT
    'POST_03_COLUMN_<COLNAME>_EXISTS' AS CHECKNAME,
    CASE
        WHEN COUNT(*) > 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'Column <COLNAME> exists in <SCHEMA>.<TABLE>' AS EXPECTED,
    COUNT(*) || ' matches found' AS ACTUAL
FROM
    IDENTIFIER($V_O_INFO_COLS)
WHERE
    TABLE_SCHEMA = '<SCHEMA>'
    AND TABLE_NAME = '<TABLE>'
    AND COLUMN_NAME = '<COLNAME>'
```

### Row Count Check
```sql
SELECT
    'POST_04_TABLE_HAS_DATA' AS CHECKNAME,
    CASE
        WHEN ROW_COUNT > 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'Table has rows after deployment' AS EXPECTED,
    ROW_COUNT || ' rows loaded' AS ACTUAL
FROM
(
    SELECT COUNT(*) AS ROW_COUNT
    FROM IDENTIFIER($V_O_DW_TABLE)
)
```

### Duplicate Check
```sql
SELECT
    'POST_05_NO_DUPLICATES' AS CHECKNAME,
    CASE
        WHEN DUP_COUNT = 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'No duplicate <KEY> values in active set' AS EXPECTED,
    DUP_COUNT || ' duplicates'
        || CASE WHEN DUP_COUNT > 0
            THEN ': ' || DUP_LIST
            ELSE '' END AS ACTUAL
FROM
(
    SELECT
        COUNT(*) AS DUP_COUNT,
        COALESCE(LISTAGG(CAST(<KEY> AS VARCHAR), ', ') WITHIN GROUP (ORDER BY <KEY>), '') AS DUP_LIST
    FROM
    (
        SELECT <KEY>
        FROM IDENTIFIER($V_O_DW_TABLE)
        WHERE ISACTIVE = TRUE
        GROUP BY <KEY>
        HAVING COUNT(*) > 1
    )
)
```

### Backfill NULL Remaining Check
```sql
SELECT
    'POST_06_BACKFILL_COMPLETE' AS CHECKNAME,
    CASE
        WHEN NULL_COUNT = 0 THEN 'PASS'
        ELSE 'FAIL'
    END AS RESULT,
    'No remaining NULL values in <COLNAME> within backfill scope' AS EXPECTED,
    NULL_COUNT || ' NULLs remaining' AS ACTUAL
FROM
(
    SELECT COUNT(*) AS NULL_COUNT
    FROM IDENTIFIER($V_O_DW_TABLE)
    WHERE
        <COLNAME> IS NULL
        AND TO_DATE(<DATE_COL>) >= '<backfill_start_date>'
)
```

---

## Rules

1. **Single query.** All checks combined with UNION ALL. The engineer runs one query and sees the full picture.
2. **No inline IDENTIFIER() concatenation.** All database-qualified references go through pre-built variables.
3. **ACTUAL must diagnose.** On FAIL, the ACTUAL column must contain enough detail to understand what went wrong without running additional queries. Include counts, lists of offending values, column names found vs expected.
4. **Number checks sequentially.** PRE_01, PRE_02, POST_01, POST_02, etc.
5. **All SQL through the formatter.** Invoke sql-formatter-skill before outputting the final script.
6. **Match the deployment.** Only include checks relevant to what the RUN.sql actually does. Do not generate generic checks for objects the deployment does not touch.
7. **Environment routing block is mandatory.** Same 4-database block as RUN.sql and ROLLBACK.sql.

---

## Output

Deliver the MONITORING.sql as a formatted SQL code block in chat for inline review. Use `present_files` only for the final deliverable .sql file after the engineer approves.
