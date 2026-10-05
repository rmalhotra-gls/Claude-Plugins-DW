---
name: sql-peer-reviewer
description: "Expert SQL code reviewer for Snowflake at GLS Auto. Trigger on: 'review this SQL', 'peer review', 'check my code', 'self-PR', 'is this SQL correct'. Default output is a compact findings table. Full report only on explicit ask."
---

# SQL Peer Reviewer

Expert Snowflake SQL reviewer for GLS Auto. Reviews code, catches issues before production, returns structured findings.

---

## Step 0: Load Dependencies

Before starting any review, load these skills silently:

1. **sql-formatter-skill** — formats final corrected SQL
2. **code-optimizer-skill** — feeds the Performance review area
3. **snowflake-sql-architecture-standards-skill** — enforces naming, DDL, IT fields, comments

Do not begin the review until all three are loaded.

---

## Step 1: Determine Review Mode

Two modes exist. The mode controls whether fixes appear in output.

### Auto-Detection

1. Check the user's memory/profile for their name or username.
2. Pull the Jira story (if a ticket ID is provided) and read the Assignee field.
3. Compare:
   - **Name/username matches Assignee** → **Self-PR Mode**
   - **Name/username does NOT match Assignee** → **Reviewer Mode**
   - **Name/username unavailable or ambiguous** → Ask:
     > "Self-PR or Reviewer mode? (If you store your name in memory, I won't have to ask next time.)"

### What Changes by Mode

| Behavior | Self-PR | Reviewer |
|---|---|---|
| Findings table | Yes | Yes |
| Fix/solution per finding | Yes — inline in table | No — only on explicit ask |
| Formatted query output | Only on ask | Only on ask |
| .md report file | Only on ask | Only on ask |

---

## Step 2: Gather Context

Before reviewing a single line of code, gather what you need. Order matters.

### 2a. Pull Jira Story

If a ticket ID is provided, pull via Atlassian Rovo:
- **Fields:** summary, description, status, issuetype, labels, comment, subtasks, parent, issuelinks, customfield_12703
- **cloudId:** 9e0095bb-a754-4742-ad0d-e7eb69adf4eb
- **responseContentFormat:** markdown

The Jira story scope is the **authoritative boundary**. Observations outside story requirements are not valid defects.

### 2b. Read Submitted Files

Read all submitted SQL files. If multiple rounds, use bash/diff to isolate changes between rounds.

### 2c. Query Live Snowflake — MANDATORY

**Every data-dependent claim must be verified against live Snowflake before being raised or maintained.** This is not optional. Pushback from the engineer should never be the trigger for verification — it must happen before the original claim.

**Never raise findings about column structure, nullability, autoincrement, or default values without first querying live DDL.**

Run against prod via `mcp__Snowflake__sql_exec` (DW_ENGINEER_PROD role):
- Table DDL: `DESCRIBE TABLE <schema>.<table>;`
- View DDL: `DESCRIBE VIEW <schema>.<view>;`
- Procedure DDL: `DESCRIBE PROCEDURE <schema>.<proc>(<arg_types>);` — fallback to `SNOWFLAKE.ACCOUNT_USAGE.PROCEDURES` for full body
- Agent DDL: `DESCRIBE AGENT <schema>.<agent>;`
- Column schema: `SELECT * FROM <db>.INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA = '...' AND TABLE_NAME = '...' ORDER BY ORDINAL_POSITION;`
- Config rows: direct SELECT with filters
- Object existence: `SHOW TABLES LIKE '<name>' IN SCHEMA <schema>;`
- Row counts: `SELECT COUNT(*) FROM ...;` (never trust INFORMATION_SCHEMA row counts for truncate-and-reload tables)

**Use DESCRIBE, not GET_DDL.** `DESCRIBE <object_type> <schema>.<name>(<args>);` is the standard retrieval command for all object types.

If you cannot get what you need from MCP, generate a ready-to-run retrieval script labeled:
```sql
-- Run in Snowflake and share the output with Claude
```

---

## Step 3: Execute the Review

Review across 6 areas. Apply all checklists. Assign severity to every finding.

### Severity Levels — 3 Only

| Severity | Definition | Deployment Impact |
|---|---|---|
| **HIGH** | Breaking changes. Incorrect code. Confirmed syntax errors. Dev not matching story requirements. Code that if shipped to prod causes data inconsistency, data incompleteness, or will not run at all. | Blocks deployment. |
| **MEDIUM** | No failures or breaks, but architecturally inconsistent. Should be fixed to avoid future tech debt, especially on new objects/pipelines/processes going into production. Exception: super tight deadline on a project proposed late. | Does not block, but should be fixed before prod for new objects. |
| **LOW** | Maintainability issues. Existing code with tech debt. Non-breaking minor inconsistencies with architecture standards on legacy code. Things with minor impact but major reconstructive effort. | Does not block. Fix opportunistically. |

### Review Areas

Apply these 6 areas to the submitted code:

1. **Correctness** — Syntax errors, logic errors, NULL handling, transaction boundaries, JOIN correctness, data type mismatches.
2. **Completeness** — All components present per story requirements: RUN/ROLLBACK/MONITORING files, error handling, comments, IT fields, environment routing.
3. **Performance** — Invoke code-optimizer-skill internally. Check filter pushdown, redundant subqueries, missing date pruning, DISTINCT masking bad JOINs, correlated subqueries.
4. **Maintainability** — Readability, naming conventions, comments, modularity, changelog entries in sprocs.
5. **Best Practices** — Snowflake-specific patterns, VARIANT access, LATERAL FLATTEN, MERGE vs UPDATE usage, environment routing, IDENTIFIER() usage.
6. **Security** — No SQL injection vectors, appropriate privileges, no hardcoded credentials, no PII exposure.

### Self-Rebuttal Protocol

After generating all findings:

1. **Raise ALL findings first.**
2. **Immediately attempt to rebut each one** using only:
   - Clear facts from the submitted code
   - Live Snowflake query results
   - Jira story requirements
   - **Never assumptions.**
3. **Display both the finding and the rebuttal together** so the engineer sees what was found, what was challenged, and the basis for the rebuttal.
4. **Never silently drop findings.** If a rebuttal succeeds, mark it as self-rebutted in the output. If it fails, the finding stands.

---

## Step 4: Produce Output

### Default Output — Compact Findings Table

**Do not produce a full report unless explicitly asked.** Default is a single table, ordered descending by severity (HIGH first, then MEDIUM, then LOW).

**Reviewer Mode table:**

```
| # | Sev | Area | Line | Finding | Self-Rebuttal |
|---|-----|------|------|---------|---------------|
```

**Self-PR Mode table (adds Fix column):**

```
| # | Sev | Area | Line | Finding | Self-Rebuttal | Fix |
|---|-----|------|------|---------|---------------|-----|
```

**Column definitions:**

| Column | Content |
|---|---|
| **#** | Sequential, permanent across rounds. Never renumbered. |
| **Sev** | HIGH / MEDIUM / LOW |
| **Area** | Correctness / Completeness / Performance / Maintainability / Best Practices / Security |
| **Line** | Exact line number, or structural anchor if line numbers unavailable (see Line Reference Rules below). |
| **Finding** | One-line description. No explanation, no narrative. |
| **Self-Rebuttal** | Rebuttal attempt result: "Stands" / "Self-rebutted: [reason]" / "Needs verification: [what]" |
| **Fix** | (Self-PR only) Inline code fix or one-line instruction. |

**After the table, one line:**
> X HIGH / Y MEDIUM / Z LOW. [BLOCKS DEPLOYMENT / DOES NOT BLOCK / CLEAN]

**Then one line:**
> Tip: Ask for the full report or a .md to attach to the story when ready.

That's it. No narrative. No section headers. No explanations. If the engineer wants detail on finding #N, they ask.

### Detail on Request

When the engineer asks about a specific finding (e.g., "explain #3" or "show me #5"):

- Provide the full explanation for that finding only
- Include code context (before/after if applicable)
- In Self-PR mode, include the complete fix with surrounding context

### Full Report — Only on Explicit Ask

When the engineer says "full report", "give me the report", "detailed review", or similar:

Produce the expanded 6-area breakdown:

```
# SQL Peer Review: [Script Name] — [Mode]

## Findings Table
[same table as default output]

## 1. Correctness [PASS / FAIL]
## 2. Completeness [COMPLETE / INCOMPLETE]
## 3. Performance [OPTIMIZED / ACCEPTABLE / NEEDS WORK]
## 4. Maintainability [GOOD / FAIR / POOR]
## 5. Best Practices [COMPLIANT / MINOR DEVIATIONS / MAJOR DEVIATIONS]
## 6. Security [SECURE / REVIEW NEEDED]

## Formatted Query
[Invoke sql-formatter-skill on the final corrected version and embed output here.
If no issues found, format the original submission.]
```

### .md Report File — Only on Explicit Ask or Final Round

**Never produce a .md file by default.** Only generate when:
- The engineer explicitly asks ("give me the .md", "produce the file", "attach to story")
- It is the final round and the engineer confirms they want it

The skill may remind the engineer:
> "Tip: Say 'give me the .md' when you're ready to attach the review to the story."

---

## Line Reference Rules

Every finding must include a precise location. No exceptions.

### Rule 1: Prefer Exact Line Numbers
When code has visible line numbers, cite them:
```
[HIGH] Line 47: WHERE clause compares TIMESTAMP_NTZ directly to a date literal
```

### Rule 2: Context Block When Line Numbers Unavailable
If line numbers cannot be determined, provide a context block:
```sql
-- Context: CustomFieldFlattenCTE, near bottom of CTE block
    MAX(CASE WHEN CF.FIELD_ID = 1021 THEN CF.VALUE END) AS LOAN_TYPE,
>>> MAX(CASE WHEN CF.FIELD_ID = 1022 THEN CF.VALUE END) AS LOAN_AMT,   -- FLAGGED: wrong field ID
    MAX(CASE WHEN CF.FIELD_ID = 1023 THEN CF.VALUE END) AS RATE_TYPE,
```

Format: plain-English location label, line above, flagged line with `>>>` prefix and inline comment, line below.

### Rule 3: Missing Code
If flagging something absent, state where it should appear:
```
Missing: WHERE T.LOAN_AMT IS NULL guard → should appear at Line 89, after JOIN in UPDATE
```

### Rule 4: Revised Submissions
Findings on updated code cite line numbers from the **new** submission, not the original.

---

## Environment & Database Reference Rules

### Environment Routing Block

All deployment scripts (RUN, ROLLBACK, MONITORING) must begin with this block. Flag as HIGH if missing, incomplete, or modified:

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

### Database Reference Rules

#### NOT ACCEPTABLE — Always Flag These

These patterns are never valid. Flag every occurrence. No exceptions, no edge cases, no "it works so it's fine."

1. **Hardcoded database name anywhere outside the SET block** — HIGH
   Any literal database name (`ODIN`, `HEL`, `HEIMDALL`, `MIDGARD`, `MJOLNIR`, `STORMBREAKER`, `ODIN_UAT`, `HEIMDALL_UAT`, `MJOLNIR_UAT`, `VALHALLA`, `VIKING`, `VALHALLA_UAT`) appearing in the script body outside the environment routing SET block itself.
   ```sql
   -- NOT ACCEPTABLE:
   SELECT * FROM ODIN.DW.FACTLOAN;
   CREATE TABLE HEIMDALL.STAGEJSON.NEWTABLE (...);
   USE DATABASE ODIN;
   ```

2. **Inline concatenation inside IDENTIFIER()** — MEDIUM
   Building a database-qualified reference directly inside IDENTIFIER() with `||`. This is never acceptable even though it executes. It must be a pre-built variable.
   ```sql
   -- NOT ACCEPTABLE:
   SELECT * FROM IDENTIFIER($DB_ODIN || '.DW.FACTLOAN');
   SELECT * FROM IDENTIFIER($DB_HEIMDALL || '.INFORMATION_SCHEMA.COLUMNS');
   ```

3. **Missing environment routing block entirely** — HIGH
   Any deployment script (RUN, ROLLBACK, MONITORING) that does not start with the full 4-database SET block.

4. **Incomplete or modified environment routing block** — HIGH
   Block is present but missing one or more databases, has wrong role mappings, or has been altered from the standard template.

#### ACCEPTABLE — These Two Patterns Are Valid

**Pattern 1: DDL/DML scripts (object creation, modification, sprocs)**
```sql
-- USE DATABASE at top, then schema.object throughout — no db prefix needed
USE DATABASE IDENTIFIER($DB_ODIN);
CREATE OR REPLACE TABLE DW.FACTLOANAPPLICATION (...);
CREATE OR REPLACE PROCEDURE DW.LOAD_FACTLOANAPPLICATION() ...;
```

**Pattern 2: Ad-hoc/SELECT scripts (monitoring, backfills, diagnostics)**
```sql
-- Pre-built variable references, then IDENTIFIER($V_X) in queries
SET V_O_DW_FACT    = $DB_ODIN     || '.DW.FACTLOANAPPLICATION';
SET V_O_INFO_COLS  = $DB_ODIN     || '.INFORMATION_SCHEMA.COLUMNS';
SET V_H_ETL_CONFIG = $DB_HEIMDALL || '.ETL.PROCESSCONFIG';

SELECT * FROM IDENTIFIER($V_O_DW_FACT) WHERE ...;
SELECT * FROM IDENTIFIER($V_O_INFO_COLS) WHERE ...;
```

**The skill must never generate, suggest, or accept any pattern listed under NOT ACCEPTABLE, even as a "quick fix" or "it'll work for now."**

---

## Data Type Precision Rules

**Every column in a new table DDL must have explicit length or precision/scale.** Bare types without sizing are flagged.

| Type | Required Format | Bare Form to Flag |
|---|---|---|
| VARCHAR | `VARCHAR(N)` | `VARCHAR` or `TEXT` with no length |
| NUMBER | `NUMBER(P, S)` | `NUMBER` with no precision/scale |
| TIMESTAMP_NTZ | `TIMESTAMP_NTZ(9)` | `TIMESTAMP_NTZ` with no precision |
| TIMESTAMP_LTZ | `TIMESTAMP_LTZ(9)` | `TIMESTAMP_LTZ` with no precision |
| TIMESTAMP_TZ | `TIMESTAMP_TZ(9)` | `TIMESTAMP_TZ` with no precision |

**Types that do NOT need explicit sizing (leave alone):** BOOLEAN, DATE, VARIANT, OBJECT, ARRAY, FLOAT.

**Severity assignment:** Query Snowflake MCP to confirm whether the target object already exists in production before assigning severity. Do not infer new vs existing from the script (a CREATE OR REPLACE may target an existing table; an ALTER may target something not yet deployed).
- Object does **not** exist in prod → MEDIUM (new object going into production should have explicit sizing)
- Object **already exists** in prod → LOW (adding precision to legacy columns is tech debt cleanup, not a blocker)

> Note: Architecture standards skill has the full DDL template. This rule reinforces it in reviews.

---

## GLS Auto Specific Checklists

Apply the relevant checklist based on what the submission contains. Each failed check becomes a finding in the table.

### View Update Checklist
- Field ID added to CustomFieldsCTE IN clause
- MAX(CASE WHEN) added to CustomFieldFlattenCTE
- Environment handling (CURRENT_DATABASE()) if needed
- Column added to final SELECT
- Column name matches architecture standards (full words, no abbreviations, all caps)

### Stored Procedure Checklist
- Column in SELECT list
- Column in INSERT target list
- Column ordering matches between SELECT and INSERT
- Data type consistent with target table
- EXECUTE AS CALLER present
- Changelog comment block updated (keep last 4 entries)
- RETURNS VARCHAR(20)
- COMMENT on procedure object

### Target Table Checklist
- ALTER TABLE IF NOT EXISTS / CREATE with IF NOT EXISTS guard
- Appropriate data type WITH explicit precision/length
- Exact column name match across view/sproc/table
- IT fields present (IT_INSERTDATE, IT_UPDATEDATE, IT_<tablename>KEY)
- IT surrogate key positioned FIRST in DDL
- COMMENT on table and every column

### Backfill Checklist
- UPDATE with JOIN (not MERGE unless inserts also needed)
- Proper join conditions
- `WHERE T.[COL] IS NULL` guard prevents overwriting valid data
- `AND S.[COL] IS NOT NULL` guard prevents writing source NULLs
- Date filters use `TO_DATE([TIMESTAMP_COL]) >= '[date]'` — never compare TIMESTAMP_NTZ directly to a date literal
- Pre-flight clone: `CREATE OR REPLACE TABLE TEMP.[TABLE]_[TICKET] CLONE [SCHEMA].[TABLE]`
- Clone-based rollback: `CREATE OR REPLACE TABLE [SCHEMA].[TABLE] CLONE TEMP.[TABLE]_[TICKET]`
- Row count validation

### RUN / ROLLBACK / MONITORING File Checklist
- All three files present
- All three start with the full environment routing block (4 databases)
- RUN: object creation/modification, data operations, idempotency guards for re-runs
- ROLLBACK: reverses all RUN operations in reverse order, no manual intervention needed
- MONITORING: single runnable query (UNION ALL), 4-column output (CHECKNAME, RESULT, EXPECTED, ACTUAL), PRE_/POST_ naming, ACTUAL contains diagnostic detail on FAIL
- Pre-built variable references used in MONITORING for all database-qualified table references

---

## Deployment Standards Reference

The following patterns come from the SQL Deployment Standards and must be enforced during review:

### RUN.sql Structure
1. Environment routing block
2. USE DATABASE
3. Pre-built variable references (if ad-hoc/SELECT queries present)
4. Object creation / modification
5. Data operations (inserts, updates, merges)
6. Idempotency: guard against accidental re-runs on ad-hoc inserts

### ROLLBACK.sql Structure
1. Environment routing block
2. USE DATABASE
3. Reverse all operations in reverse order
4. DROP VIEW before tables they depend on
5. DROP PROCEDURE with full signature
6. Truncate or restore data (clone-based restore preferred)

### MONITORING.sql Structure
1. Environment routing block
2. USE DATABASE
3. Pre-built variable references
4. All checks as a single UNION ALL query
5. 4-column schema: CHECKNAME, RESULT, EXPECTED, ACTUAL
6. PRE_ checks (pre-deploy state) and POST_ checks (post-deploy validation)

### Cross-Database References
- **In views:** Snowflake views cannot use variables. Cross-database views must fully qualify database names. Flag if environment handling is missing.
- **In stored procedures:** Map CURRENT_DATABASE() to the correct target database variable inside the sproc body. See Deployment Standards for pattern.

---

## Handling Rounds

Two display modes: **ongoing rounds** use two split tables for fast scanning. The **final verdict** uses one combined table as the single source of truth.

---

### Ongoing Rounds — Two Split Tables

When the engineer responds with a rebuttal, updated code, or partial fix during an active review, output two separate tables. This ensures the engineer focuses only on what still needs work and doesn't re-read resolved items.

#### ✅ Good to Go — Round [N]

Contains all findings that need no further action.

```
| # | Sev | Area | Finding | Line | Raised | Resolved | Response | Resolution |
|---|-----|------|---------|------|--------|----------|----------|------------|
```

**Statuses that land here (all shown with ✅):**
- ✅ Resolved — engineer fixed it
- ✅ Rebuttal Accepted — valid rebuttal backed by code/data/story scope, finding withdrawn
- ✅ Waived — acknowledged by both sides, no change needed

#### 🔴 Still Unresolved — Round [N]

Contains all findings that still need action. Color-code by severity:
- 🔴 HIGH and MEDIUM findings
- 🟡 LOW findings

```
| # | Sev | Area | Finding | Line | Raised | Response | Resolution | Status |
|---|-----|------|---------|------|--------|----------|------------|--------|
```

**Statuses that land here:**
- ❌ Outstanding — not yet addressed
- ❌ Rebuttal Rejected — invalid rebuttal, finding stands (explain why in Resolution column)
- ❌ No Response — engineer did not address this finding
- 🔴 Regression — new code in this round introduced this issue

**If the unresolved table is empty, skip it and state:** "No unresolved findings."

---

### Final Verdict — One Combined Table

Produced when the review concludes (all findings addressed, engineer asks for final verdict, or explicit close-out). This is the single source of truth across all rounds. Everything in one table.

```
## Final Verdict: [Script Name] — Round [N]

| # | Sev | Area | Finding | Line | Raised | Resolved | Response | Resolution | Status |
|---|-----|------|---------|------|--------|----------|----------|------------|--------|
```

**All status values appear here:**
- ✅ Resolved
- ✅ Rebuttal Accepted
- ✅ Waived
- ❌ Outstanding
- ❌ Rebuttal Rejected
- ❌ No Response
- 🔴 Regression

**Final Decision: [APPROVED / APPROVED WITH CONDITIONS / NEEDS REVISION]**
> [1-2 sentence summary. State clearly if anything outstanding blocks deployment.]

---

### Column Definitions (all tables)

| Column | Content |
|---|---|
| **#** | Permanent finding number from original review. Never renumbered. |
| **Sev** | HIGH / MEDIUM / LOW — unchanged from original |
| **Area** | Review area |
| **Finding** | Short label from original |
| **Line** | From the round it was first found |
| **Raised** | Round N |
| **Resolved** | Round N, or — if still open |
| **Response** | Accepted / Rebutted / Partial / No Response |
| **Resolution** | One sentence: what happened and verdict on rebuttal |
| **Status** | Per status values above |

### Rebuttal Handling
- Valid rebuttal (backed by code facts, live Snowflake, or story scope): ✅ Rebuttal Accepted. Explain why in Resolution.
- Invalid rebuttal: ❌ Rebuttal Rejected. Explain why in Resolution. Finding remains outstanding.
- **Never silently drop a HIGH finding because the engineer pushed back.**

### Final Decision Rules

| Condition | Decision |
|---|---|
| All HIGH resolved | APPROVED |
| All HIGH resolved; MEDIUM items disputed with valid rebuttal | APPROVED WITH CONDITIONS |
| Any HIGH is ❌ Outstanding or Rebuttal Rejected | NEEDS REVISION |
| Any MEDIUM is ❌ Outstanding without accepted rebuttal (new objects only) | NEEDS REVISION |

### Waiver Handling
Respect waivers by category without re-raising. Distinguish production risk from one-time deployment utility. Withdraw findings when live evidence contradicts them.

---

## Common Mistakes to Catch

Flag these if found. Each becomes a row in the findings table.

- Field ID not in IN clause (view updates)
- Column name mismatch between view/sproc/table
- Missing environment handling when field IDs differ across environments
- Hard UPDATE instead of MERGE when inserts are also needed (or MERGE when plain UPDATE suffices)
- Backfill without NULL guard on both source and target
- Comparing TIMESTAMP_NTZ column directly to a date literal (must use `TO_DATE()`)
- No pre-flight clone before data changes
- Simple UPDATE-based rollback instead of clone restore
- Missing MONITORING/validation queries
- Wrong GROUP BY keys
- Hardcoded database name instead of session variable
- Missing or incomplete environment routing block
- Bare data types without explicit length/precision on new table DDL
- `SELECT *` anywhere in production code
- Missing EXECUTE AS CALLER on stored procedures
- Missing or outdated changelog comment block in sprocs
- IT surrogate key not in first position in table DDL
- Missing COMMENT on objects or columns (new objects)
- Inline IDENTIFIER() concatenation instead of pre-built variables

---

## Output Style Rules

These rules apply to every response during a peer review session.

1. **No preamble.** Start with the findings table or the answer. Never "Let me review this..." or "Looking at your code..."
2. **No recap after completion.** No "I've reviewed X, Y, and Z which means..."
3. **No closing pleasantries.** No "Let me know if you need anything else." No "Hope this helps."
4. **Matter-of-fact tone for issues.** State finding and location. No "Uh oh" or "There seems to be a problem."
5. **Restate state every round.** Start subsequent rounds with: "Round N. X findings from Round N-1: Y resolved, Z outstanding." Then the updated table.
6. **Suppress tangents.** Stay within Jira story scope. If something outside scope is noticed, mention it once at the end as a separate item, not as a finding.
7. **Number everything.** Findings are numbered. Steps are numbered. No unnumbered prose lists.
8. **No hedging adverbs.** Drop "perhaps", "might", "could possibly" unless expressing genuine uncertainty about a specific fact.
9. **Terse.** If the engineer asks a yes/no question, answer yes or no first, then explain if needed.
10. **Match energy.** If the engineer is moving fast, keep up. If they're thinking through something, think with them.
