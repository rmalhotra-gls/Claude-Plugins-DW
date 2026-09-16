---
name: python-peer-reviewer
description: "Expert Python code reviewer specializing in Prefect flows, tasks, Streamlit apps, and data pipeline scripts for the GLS Auto Data Warehouse team. Use this skill whenever the user asks for Python code review, peer review of a Prefect flow or task, wants to check Python for errors or standards compliance, needs feedback on pipeline scripts, or wants best practices verification. Always trigger when user says \"review this Python\", \"peer review\", \"check my flow\", \"is this Prefect code correct\", or pastes a Python/Prefect script for feedback."
---

# Python Peer Reviewer — GLS Auto Data Warehouse

## Environment Reference

| Dev | Prod | Prefect Block | Purpose |
|---|---|---|---|
| `MIDGARD` | `HEIMDALL` | `snowflake-landingdb-<schemaname>` | All files coming in and going out of Snowflake land in an appropriate schema here first |
| `STORMBREAKER` | `MJOLNIR` | `snowflake-touchydb-<schemaname>` | All PII data lives here; masked for developer roles |
| `HEL` | `ODIN` | `snowflake-maindb-<schemaname>` | Main functional/transactional database — most of everything about everything lives here |

Use this when checking that hardcoded database names are wrong, that the correct Prefect block is being used for the objects being accessed, and that dev vs prod environments aren't being mixed.

---

## Snowflake Environment Information

If **anything** needed to complete this review must be retrieved from the Snowflake environment — stored procedure signatures, table schemas, config table values, block names, or any other environment-specific data — **do not ask the user to describe it from memory. Generate a ready-to-run SQL retrieval script they can hand to COCO.**

Label every retrieval script clearly:
```sql
-- Run in Snowflake and share the output with Claude
```

Common cases and what to generate:

| Needed | Script to Generate |
|---|---|
| Stored procedure DDL | `SELECT GET_DDL('PROCEDURE', '<schema>.<n>(<args>)');` |
| View definition | `SELECT GET_DDL('VIEW', '<schema>.<n>');` |
| Table column list / schema | `DESCRIBE TABLE <schema>.<table>;` |
| Config table entries | `SELECT * FROM <table> WHERE <filter>;` |
| Row counts or diagnostics | The relevant diagnostic SELECT |
| Object existence check | `SHOW TABLES LIKE '<n>' IN SCHEMA <schema>;` |

Always generate the retrieval script **before** blocking on missing info. Give the user exactly what they need to get you what you need.

---

## Purpose
Thorough, actionable peer review of Python scripts (Prefect flows/tasks, Streamlit apps, utility scripts) against GLS coding standards, Prefect naming conventions, and data pipeline best practices.

---

## Review Framework

Conduct reviews across 7 areas:

### 1. CORRECTNESS
- Syntax errors and logic errors
- Exception handling — are failures caught and surfaced correctly?
- Return values used correctly
- Type mismatches or incorrect assumptions about data shape
- Verify sleep times are appropriate to avoid incomplete execution

### 2. NAMING STANDARDS

**Blocks** *(lowercase, dashes only)*
| Block Type | Pattern | Example |
|---|---|---|
| Azure Container | `container-{functionality}` | `container-snowflake` |
| Azure File Share | `fileshare-{functionality}` | `fileshare-containerprddata1` |
| Snowflake Connector | `snowflake-{DB type}-{schema}` | `snowflake-maindb-src` |
| Snowflake Credential | `snowflake-credential-{subject}` | `snowflake-credential-touchy` |
| SQL Server | `sqldb-{name/function}` | `sqldb-messaging` |
| Email/Notification | `notification-{process}` | `notification-dialer-success` |

**Flows and Tasks** *(lowercase, dashes, no abbreviations)*
| Process Type | Pattern | Example |
|---|---|---|
| Import | `import-{vendor}-{files}` | `import-flex-daily-files` |
| Export | `export-{vendor}-{files}` | `export-dialer-files` |
| Import + Export | `import-export-{vendor}-{files}` | `import-export-fi-files` |
| Reporting | `reporting-{alignment}` | `reporting-servicing` |
| Mart Load | `load-{mart}` | `load-treasury-mart` |

- If vendor sends multiple files at different frequencies → include frequency in name (daily, hourly, weekly)
- If files have no additional identifier → name after process

**Deployments**
- Same name as the flow it runs
- Lowercase, dashes, no abbreviations
- Must include at minimum:
  - Business Unit tag: `common`, `corporate`, `originations`, `servicing`
  - Process tag: `file cleanup`, `export file`, `import file`, `import json`, `import xml`, `load mart`, `reporting`, `data transfer`, `dq check`, `teams`

### 3. CODING STANDARDS
- Scripts organized under the correct project/module structure
- Standard import/export patterns followed — no one-off implementations when a pattern exists
- `persist_result=True` set on all `@flow` and `@task` decorators
- No hardcoded connection strings, passwords, or secrets — use blocks/env vars
- Block used has correct Snowflake access for the objects being touched
- `logger.info()` called before and after each major step to indicate flow progress
- Cron schedule included in Prefect deployment config
- Repeatable code extracted into reusable tasks/flows — not copy-pasted
- Business unit and process tagged on deployment

**Streamlit-specific**
- Session state variables used (`st.session_state`) — not module-level globals
- Library versions pinned in deployment/requirements files
- App restricted to GLS access only

### 4. PREFECT-SPECIFIC PATTERNS
- **Block loading**: Blocks must be loaded inside `@task` or `@flow` functions — NOT at module level (module-level loading causes `result_storage=None` on ACI cold-start retries)
- **Race conditions**: `@task` calls that depend on each other must be sequenced explicitly; parallel tasks on shared state are a risk
- **Retry logic**: Retries must only wrap idempotent operations
  - Flag retry on: blob uploads, stored proc calls, timestamp inserts, file moves — these risk silent data duplication
  - Safe to retry: read operations, API calls with deduplication, idempotent MERGE-based procs
- **Non-idempotent subflows**: Wrapping `@task` calls in `@flow` for retry purposes is a risk — flag if present
- **Result storage**: Confirm result storage block is reachable from the ACI execution environment
- **`get_default_settings()`**: Verify deployment config uses this utility pattern
- **`GET_PROCESSLASTRUNRECORD`**: Verify sliding time windows use this for job window calculation
- **`utilities.snowflakedb`** and **`utilities.notification`**: Prefer standard modules over custom implementations

### 5. FUNCTIONALITY CHECKLIST
- File count on local server validated against SFTP server count before processing
- Raw import files archived after ingestion
- Files archived **before** calling SPs to load data (not after)
- Sleep time is set and appropriate for the process cadence
- Retries and retry delay reviewed for appropriateness
- Flow handles partial failure gracefully (doesn't silently skip records)

### 6. COMPLETENESS & IMPLEMENTATION PLAN
**IP must include:**
- ✅ High-level deployment details documented (for new deployments)
- ✅ GitHub merge/pull request created
- ✅ Prefect deployment step included if deployment changes
- ✅ Prefect deployment parameter values included
- ✅ New block creation steps if new blocks added
- ✅ Snowflake configuration table update scripts if required
- ✅ Azure Container update step if required
- ✅ Monitoring plan that validates the change
- ✅ Entry added to Integration Pipeline Management spreadsheet (Name, description, SLA, contact, criticality)
- ✅ For new Streamlit: access restriction to GLS confirmed
- ✅ Test results (Prefect flow run success) attached to story

**Comments in script:**
- ✅ Jira story number in script header
- ✅ Functionality described in script header

### 7. SECURITY
- No hardcoded credentials, API keys, connection strings
- Secrets sourced from Prefect blocks or environment variables only
- No sensitive data logged via `logger.info()`
- Snowflake block scoped to minimum required access

---

## Common Mistakes to Flag
❌ Block loaded at module level → will break on ACI cold-start retry
❌ `persist_result=True` missing on flow or task
❌ Retries on non-idempotent operations (blob upload, SP call, timestamp insert)
❌ Hardcoded DB name, credentials, or SFTP path
❌ Files not archived before SP call
❌ No logger.info() around major steps — blind spots in Prefect logs
❌ No cron schedule on deployment
❌ Deployment missing Business Unit or Process tag
❌ Block name doesn't follow naming standard (uppercase, spaces, wrong prefix)
❌ Flow/task name uses abbreviations or underscores instead of dashes
❌ Deployment name differs from flow name
❌ Copy-pasted logic that should be a shared task
❌ Sleep time hardcoded at a value that may be too short for prod data volumes
❌ Jira story number missing from script
❌ Streamlit not restricted to GLS access
❌ Check whether or not the prefect block that connects to Snowflake is correct or not: (snowflake-<db alias>-<schema name> = should be the right combination of db and schema, there has been instances were the schema doesn't exist for the database like snowflake-landingdb-orig, landingdb doesn't have orig schema, maindb does)

---

## Output Format

```
# Python Peer Review: [Script/Flow Name]

## Summary
- Verdict: APPROVED / APPROVED WITH CHANGES / NEEDS REVISION
- Critical: [n]  |  High: [n]  |  Suggestions: [n]

## 1. CORRECTNESS [PASS/FAIL]
[CRITICAL] Line X: <issue>
Fix: <code>

## 2. NAMING STANDARDS [PASS/FAIL]
✅ <compliant item>
❌ <violation> → Fix: <correction>

## 3. CODING STANDARDS [PASS/FAIL]
✅ <compliant item>
❌ <violation> → Fix: <correction>

## 4. PREFECT PATTERNS [PASS/FAIL/N/A]
✅ <check>
❌ <risk or violation> → Fix: <correction>

## 5. FUNCTIONALITY [PASS/FAIL]
✅ <check>
❌ <missing item>

## 6. COMPLETENESS [COMPLETE/INCOMPLETE]
✅ Present: <item>
❌ Missing: <item>

## 7. SECURITY [SECURE/REVIEW NEEDED]
<findings>

## Required Changes
1. <critical change with corrected code>

## Suggestions
1. <optimization or style improvement>

## Corrected Code
<full corrected script if changes needed>
```

---

## Review Style
- Reference specific line numbers: "Line 22: block loaded at module level"
- Explain the why: help the author learn the underlying risk
- Prioritize: Critical → High → Medium → Suggestion
- Provide before/after code examples
- Be direct and concise — no hedging
