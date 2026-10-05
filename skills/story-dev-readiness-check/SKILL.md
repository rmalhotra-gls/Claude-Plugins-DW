---
name: user-story-validator-skill
description: "Validates Jira user stories and acceptance criteria for engineering readiness. Use this skill whenever the user pastes a user story, acceptance criteria, Jira ticket content, or asks to review a story for completeness, logic gaps, or readiness. Always trigger when user says 'review this story', 'is this ready for dev', 'check this ticket', 'validate this AC', 'story review', 'definition of ready', 'DoR check', or pastes anything that looks like a user story or acceptance criteria. Also trigger when user mentions BSA handoff, sprint planning prep, backlog grooming review, or story quality check."
---

# User Story Validator — Definition of Ready Enforcer

## Purpose

Gate user stories before they enter a sprint. The test is simple:

> **Can an entry-level engineer read this story and immediately answer two questions?**
> 1. **What does DONE look like?** — Can they picture the end state with zero ambiguity?
> 2. **How big is this?** — Can they estimate story points without asking anyone anything?

If yes to both → READY FOR DEV.
If no to either → NOT READY. Back to the BSA with a specific list of what's missing.

An entry-level engineer CAN:
- Read architecture standards and follow them (table naming, IT fields, SCD patterns)
- Ask a senior engineer for architecture guidance and get an answer
- Write SQL, build sprocs, design tables — figure out the HOW
- Run queries to explore data and verify assumptions
- Estimate effort once scope is clear

An entry-level engineer CANNOT:
- Guess what the business means by vague terms ("active dealer", "recent data")
- Know that the override logic lives in Stephanie's head and not in the story
- Discover that two tables with similar names have different column names for the same concept
- Infer that changing line 42 also requires changing the WHERE clause on line 45
- Know that "a later story" with no ticket number means a dependency that doesn't exist yet
- Know who to escalate to when the AC seems disproportionate to the data

**Everything the engineer can't figure out from the codebase + architecture standards must be in the story.**

---

## Guiding Principles

> The story is a **work order**, not a conversation starter.
> If an engineer has to ask a question to start coding, the story is not ready.

> **The story covers WHAT, WHERE, and WHEN. Never HOW.**
> The BSA defines the business requirement. Engineering figures out the implementation.
> Table naming, deployment patterns, retry logic, file handling strategies, SCD mechanics,
> stored procedure design — these are all HOW. They belong to engineering.
> The story must tell engineering: what data, from where, going where, by when, and what
> the business expects to see at the end. That's it.

### The Two-Question Test (Apply to Every Dimension)

Before flagging a gap, check it against both questions:

```
Would this missing info prevent the engineer from picturing DONE?
  YES → Flag it (Critical if it blocks starting, Warning if they'd discover it mid-dev)

Would this missing info prevent the engineer from estimating EFFORT?
  YES → Flag it (Critical — you can't plan a sprint with unknowable effort)

Would this missing info only matter DURING implementation (the HOW)?
  YES → Don't flag it — engineering figures it out
```

### The BSA vs. Engineering Boundary

Use this decision tree on every potential gap before flagging it:

```
Is this about WHAT the business needs, WHERE data comes from/goes, or WHEN it happens?
  YES → BSA owns it → Flag if missing
  NO  → Is this about HOW to build it?
    YES → Engineering owns it → DO NOT flag as a story gap
    NO  → Is this about operational expectations (SLAs, contacts, escalation)?
      YES → BSA owns the business side (vendor SLA, escalation contact, tolerance window)
            Engineering owns the technical side (retry logic, alert routing, error handling)
            → Flag only the business side if missing
```

**BSA owns (flag if missing):**
- What data the business needs and why
- Where the data comes from (source system, vendor, file drop location)
- Where the data should be accessible (which teams, which reports)
- When data arrives / when it's needed by
- Business rules for the data (what "active" means, what statuses are valid, etc.)
- Vendor contacts and escalation paths
- SLAs with external parties (response time, resolution time)
- Known limitations of the source ("file can be late by X hours", "vendor doesn't send on holidays")
- What "done" looks like from the business perspective

**Engineering owns (never flag as a story gap):**
- Table names, schema names, database names (DW naming standards govern this)
- Table structure (SCD type, partitioning, clustering)
- Staging/landing zone design
- Stored procedure design and naming
- Retry logic, polling intervals, error handling patterns
- Deployment artifacts (RUN.sql, ROLLBACK.sql, VALIDATE.sql)
- File handling (duplicate files, empty files, malformed files)
- Data type casting and storage decisions
- IT fields, surrogate keys, audit columns
- Prefect flow design, task breakdown, scheduling mechanics
- Decryption implementation details (key retrieval, library choice)

---

## Input

The user will paste one of:
- A Jira story (title + description + acceptance criteria)
- Just acceptance criteria
- A requirements doc or snippet
- A screenshot or copied text from a ticket

If the input is ambiguous about what it is, ask one question: "Is this the full story or just a piece of it?"

---

## Validation Dimensions

The 12 dimensions below are grouped by which question they answer.

**"What does DONE look like?"** — These must be in the story or the engineer can't picture the end state:

| DIM | Name | What it checks |
|---|---|---|
| 1 | OBJECTS | Where does data come from, where does it go? |
| 2 | COLUMNS | What fields are involved? |
| 3 | BUSINESS LOGIC | What are the rules? |
| 5 | ACCEPTANCE CRITERIA | How does the business verify success? |
| 7 | DELIVERABLES | What does the business get at the end? |
| 8 | SCOPE BOUNDARIES | What's in and what's out? |
| 9 | TRIBAL KNOWLEDGE | Is all the logic written down or in someone's head? |
| 12 | DECISION OWNERSHIP | Who decides if there's a tradeoff? |

**"How big is this?"** — These must be in the story or the engineer can't estimate effort:

| DIM | Name | What it checks |
|---|---|---|
| 4 | OPERATIONAL BOUNDARIES | What vendor constraints affect the build? |
| 6 | DEPENDENCIES | What has to happen first / in parallel? |
| 10 | SCHEMA VERIFICATION | Do referenced objects actually match what the story says? (Surprises here blow up estimates) |
| 11 | CROSS-STORY DEPS | Are there hidden stories this one depends on? |

Note: Some dimensions serve both questions. They're listed where they have the most impact.

Run the story through **every** dimension below. Each dimension produces rows in the output table.

### DIM-1: OBJECTS — Are source and destination clear?

This dimension validates that the story identifies **what exists today and where data flows**. It does NOT validate target object naming — that's an engineering architecture standard.

**Branch logic:**

```
Is this a NEW source being ingested for the first time?
  YES → Source must be fully identified:
        - External system name (e.g., "DealerCenter SFTP")
        - File/API/feed location (e.g., "OutgoingDealerFiles folder")
        - File format and naming pattern
        - Delivery mechanism and schedule
        → Target naming is ENGINEERING'S job (skip, do not flag)
  NO  → Is this modifying an EXISTING pipeline?
    YES → Existing objects must be named:
          - Source table(s): Fully qualified `DATABASE.SCHEMA.TABLE`
          - Target table(s): Fully qualified `DATABASE.SCHEMA.TABLE`
          - Views/procs being modified: Fully qualified
    NO  → Is this a new internal object (view, report table)?
      YES → Source objects must be named. Target naming is engineering's job.
```

| Check | PASS condition | Applies when |
|---|---|---|
| External source identified | System name + location + format + schedule | New external ingestion |
| Existing source objects named | Fully qualified: `DATABASE.SCHEMA.TABLE` | Modifying existing pipeline |
| Existing target objects named | Fully qualified: `DATABASE.SCHEMA.TABLE` | Modifying existing table/view |
| Data destination described | Which team/report/system needs this data | Always |

**FAIL examples:** "update the table", "the contact table", "source system", "load into DW" (when modifying existing objects). These are not identifiable.

**NOT a fail:** "Load into Snowflake" for a net-new ingestion where target table doesn't exist yet — engineering names it per DW standards.

### DIM-2: COLUMNS — Are all fields specified?

**Branch logic:**

```
Is this ingesting a new external source?
  YES → Source column list must be provided (from vendor docs or sample file)
        Target column names and data types are ENGINEERING'S job (skip)
  NO  → Is this modifying an existing table?
    YES → Exact column names required (both existing and new)
          For new columns on existing tables: data type + nullable + default required
          (BSA must coordinate with engineering on this during grooming)
```

| Check | PASS condition | Applies when |
|---|---|---|
| Source columns listed | Exact column names from source, not descriptions | Always |
| Column descriptions provided | What each column means in business terms | New source ingestion |
| Source-to-target mapping | Which source column → which target column | Modifying existing pipeline |
| New vs. existing stated | Explicitly stated whether columns already exist or are being added | Modifying existing table |
| Data types for new columns | Type + nullable + default | Adding columns to existing table |

**FAIL examples:** "add the new fields", "include contact info", "relevant columns".

**NOT a fail:** Source column list provided without target data types for a net-new ingestion — engineering decides storage types.

### DIM-3: BUSINESS LOGIC — Are the business rules unambiguous?

This dimension validates that the **business intent** is clear. It does NOT validate implementation patterns (SCD type, merge strategy, sproc design — those are engineering decisions).

**Branch logic:**

```
Does the story describe business rules for filtering, matching, or transforming data?
  YES → Each rule must be specific enough that two engineers would implement it the same way
  NO  → Is the story "load everything as-is"?
    YES → That's a valid answer — PASS (but must be stated, not implied)
```

| Check | PASS condition |
|---|---|
| Business filters stated | Exact conditions in business terms: "only dealers with AccountLenderStatus = Active" — not "active records" |
| Business key identified | Which field(s) uniquely identify a record from the business perspective (e.g., "DCID is the unique dealer identifier from DealerCenter") |
| History requirement stated | Does the business need to see historical changes? YES/NO. (The BSA says "we need history" or "current state only" — engineering decides SCD2 vs. snapshot vs. whatever) |
| Full file vs. delta stated | Does the source send everything every time, or only changes? |
| Business rules for joins | If data needs to be combined: which business entities connect and on what business identifier |
| Aggregation intent | If summarized: what grain does the business need (daily? per dealer? per status?) |
| Dedup business rule | If source can have duplicates: which record does the business consider authoritative and why |

**FAIL examples:** "only active ones" (which status values?), "join to contacts" (on what business identifier?), "deduplicate" (keep which one and why?), "most recent" (most recent by what — business date? file date? system timestamp?).

**NOT a fail:** Story doesn't mention NULL handling, incremental load mechanics, watermark columns, or merge strategy — those are HOW, not WHAT.

### DIM-4: OPERATIONAL BOUNDARIES — Are the business-side boundaries defined?

This dimension validates the **business expectations** around edge cases. It does NOT validate engineering's technical handling (retry logic, empty file handling, duplicate file processing, error handling patterns — those are HOW).

**Branch logic:**

```
Does this story involve an external source (vendor, file drop, API)?
  YES → BSA must define:
        - Vendor contact for escalation
        - SLA expectations (how fast vendor responds to issues)
        - Known limitations the vendor has communicated
        - How late is too late (business tolerance window, not retry count)
        - What happens from a BUSINESS perspective if data doesn't arrive
  NO  → Is this modifying existing data?
    YES → BSA must define:
          - Whether historical corrections/backfills are expected
          - Business impact of re-processing
    NO  → Likely N/A for this dimension
```

| Check | PASS condition | Owner |
|---|---|---|
| Vendor/source contact | Named person + email for escalation | BSA |
| SLA with vendor | Response time + resolution time documented | BSA |
| Known source limitations | "File can be late by X hours", "vendor doesn't send on holidays", etc. | BSA |
| Business tolerance window | "If data isn't available by [TIME], [CONSEQUENCE]" | BSA |
| Backfill expectation | Does the business expect historical corrections? YES/NO | BSA |
| First-load expectation | What does the business expect to see after go-live? (all history? from today forward?) | BSA |

**NOT a fail (engineering owns these):**
- Missing file retry logic, polling intervals
- Duplicate file handling
- Empty file handling
- Schema drift detection
- Constraint violation handling
- Alert routing and channel config

Mark as `N/A` if genuinely not applicable — but justify why.

### DIM-5: ACCEPTANCE CRITERIA — Are they testable and proportionate?

| Check | PASS condition |
|---|---|
| Each AC is a single assertion | One check per AC, not "and also verify…" |
| AC has expected value | "Row count > 0" not "data loads successfully" |
| AC is automatable | Could be written as a VALIDATE.sql check (per monitoring standards) |
| AC covers happy path + failure path | At least one AC for what happens when something goes wrong |
| No subjective language | No "appropriate", "reasonable", "as expected", "correct" without a definition |
| AC proportionate to stated need | If AC requires complex logic (carry-forward, multi-scenario edge cases), the story explains why the complexity is warranted. If the BSA isn't sure, the story should say "if data shows this affects < N records, simplify to [APPROACH]" |
| Edge cases enumerated | If the AC implies edge cases exist (e.g., "handle overrides correctly"), every edge case scenario is listed — not left for the engineer to derive |

**FAIL examples:** "data loads correctly", "verify the output looks right", "ensure quality", "handle all override scenarios" (how many? which ones?).

**The "17 edge cases" test:** If an engineer would need to derive more than 3 edge case scenarios from the AC before they could write test cases, the AC is incomplete. The scenarios must be in the story.

### DIM-6: DEPENDENCIES

| Check | PASS condition | Owner |
|---|---|---|
| External system dependencies | API, file drop, SFTP, Salesforce sync — named with access details | BSA |
| Upstream data dependencies | Which business data must exist before this makes sense (e.g., "dealers must already be in DeFi") | BSA |
| Cross-team dependencies | If another team must do something first (e.g., "InfoSec must whitelist the IP") | BSA |
| Timeline dependencies | "Vendor goes live on [DATE]", "must be ready before [EVENT]" | BSA |

**NOT a fail (engineering owns these):**
- Deployment order / sequencing of scripts
- Environment targeting (DEV/TEST/UAT/PROD)
- Permissions, roles, grants
- Prefect schedule coordination

### DIM-7: DELIVERABLES — Is the business ask clear?

| Check | PASS condition |
|---|---|
| What the business gets | Stated in business terms: "a table with daily DealerCenter dealer data available for reporting" |
| Scope of delivery | Is this a one-time load, a recurring process, or a modification to an existing one? |
| Done criteria | What does the business check to confirm this is working? (Not engineering's VALIDATE.sql — the business equivalent: "I can see dealer data in PowerBI by [date]") |

**NOT a fail (engineering owns these):**
- Specific object types (table vs. view vs. proc)
- DDL actions (CREATE / ALTER / MERGE)
- Deployment artifacts (RUN.sql, ROLLBACK.sql, VALIDATE.sql)

### DIM-8: SCOPE BOUNDARIES — Is the blast radius defined?

This catches the #1 cause of mid-dev rework: story says "change X" but is silent on adjacent thing Y that sits in the same sproc, same view, or same pipeline step. Engineer either touches Y (rework) or avoids Y (feature doesn't work).

**Branch logic:**

```
Does this story modify an existing object (sproc, view, table)?
  YES → Story must define:
        - What IS in scope (explicitly)
        - What is NOT in scope (explicitly) — especially adjacent logic in the same object
        - If adjacent logic MUST change for the feature to work, that's in scope — say so
  NO  → Net-new build? Scope boundary = "everything in this story, nothing else"
        → PASS (implicit boundary is fine for greenfield)
```

| Check | PASS condition |
|---|---|
| In-scope stated | Explicit list of what this story changes |
| Out-of-scope stated | Explicit list of what this story does NOT touch, especially adjacent logic in the same object |
| Adjacent dependencies acknowledged | If changing column X requires changing WHERE clause Y in the same sproc, the story says so — not discovered mid-dev |

**FAIL examples (from real stories):**
- "Update status translation logic only" — but the delta detection WHERE clause in the same sproc also needs updating for the feature to work. Story was silent on it.
- "Expand to ROWNUMBER=2" — but didn't say whether a backfill was needed/pointless. Engineer could waste hours.
- Story changes a column in a sproc but doesn't mention the adjacent CALCULATEDWAL column on the same line. Engineer changes it, rework follows.

**What a READY version looks like:**
```
IN SCOPE: Update LENDERSTATUS translation logic in ODIN.DW.SP_LOAD_DEALERSTATUS.
          This requires expanding the WHERE clause on lines 42-45 to include 'Temp - Active'.
OUT OF SCOPE: CALCULATEDWAL logic on line 43 — do not modify.
              No backfill needed — SRC truncates hourly, historical data is not recoverable.
```

### DIM-9: TRIBAL KNOWLEDGE — Is all business logic written down?

This catches stories that *look* complete but rely on knowledge that lives in someone's head, in Teams threads, or in handwritten notes on uploaded images — not in the story text.

**Branch logic:**

```
Does the story reference complex business logic (clustering, suppression, overrides, 
dedup rules, partner groupings, special-case handling)?
  YES → Is the FULL logic defined inline in the story or in a linked, versioned document?
    YES → PASS
    NO  → Does the story say "per existing logic" or "as discussed" or "per Stephanie's email"?
      YES → FAIL — the logic must be in the story, not in someone's inbox
  NO  → Simple CRUD / straight load? → PASS
```

| Check | PASS condition |
|---|---|
| All business rules defined inline | Every rule the engineer needs is in the story text or a linked Confluence page — not in Teams, email, or verbal agreements |
| No "as discussed" references | Story doesn't point to conversations as source of truth |
| No uploaded images as sole logic source | If images contain logic (handwritten notes, screenshots of rules), the logic is also transcribed into story text |
| Edge cases enumerated | If the business logic has known edge cases (e.g., 17 override scenarios), they are listed, not implied |
| Domain-specific terms defined | If the story uses terms like "cluster window," "suppression," "partner grouping," "override type" — each is defined with exact rules, not assumed knowledge |

**FAIL examples (from real stories):**
- Cluster window type (fixed vs rolling), PB/CB swap rules, partner groupings — none in the Jira story. Required multiple Teams exchanges to define.
- Override type logic (most recent override type with MIN date, only when APPROVEDDATE IS NULL) came from handwritten notes in uploaded images.
- "DW Determined Extra Letter" counting rules — couldn't be answered from the story alone.

### DIM-10: SCHEMA VERIFICATION — Do referenced objects match reality?

**This dimension requires a live Snowflake check via MCP.** If MCP is not available, flag this dimension as "UNVERIFIABLE — engineer should validate before starting dev."

```
Does the story reference existing Snowflake objects by name?
  YES → Query INFORMATION_SCHEMA to verify:
        - Object exists
        - Column names match what the story says
        - Data types match what the story says
        - Nullable/NOT NULL matches what the story assumes
  NO  → N/A (net-new objects)
```

| Check | PASS condition |
|---|---|
| Referenced objects exist | Every DATABASE.SCHEMA.TABLE in the story resolves in Snowflake |
| Column names match | Story's column names match actual INFORMATION_SCHEMA column names (watch for ISACTIVE vs ACTIVE, DEALERGROUPCODE vs DEALERGROUP type divergences) |
| Data types match | Story's assumed types match actual types (watch for TEXT vs DATE, TIMESTAMP_NTZ vs DATE type mismatches) |
| Nullable assumptions correct | If story assumes a column is NOT NULL for join/dedup logic, verify it actually is (nullable columns cause silent duplicate rows) |
| Object hasn't been renamed | Column and table names haven't drifted from what the story documents |

**FAIL examples (from real stories):**
- DIMDEALER_CURR vs DIMDEALER_HIST have different column names for the same concept (ISACTIVE vs ACTIVE, SALESREPNAME vs REPRESENTATIVE). Story said "use dealer history" without flagging the divergence.
- CLIENTDEALERID is nullable — caused duplicate rows in production. Not documented.
- ASOFDT is TEXT with slash format in STG vs DATE in FACTFUNDING. Causes silent join failures.

### DIM-11: CROSS-STORY DEPENDENCIES — Are related stories linked and sequenced?

```
Does this story reference objects, logic, or outcomes from another story?
  YES → Is that story linked in Jira with a dependency relationship?
    YES → Is the dependency story DONE or at least in the same sprint?
      YES → PASS
      NO  → FAIL — this story can't be completed without the other one
    NO  → FAIL — unlinked dependency is invisible to sprint planning
  NO  → Does the story defer anything to "a later story"?
    YES → Is there a ticket number for the deferred work?
      YES → PASS (acknowledged deferral)
      NO  → FAIL — "later story" with no ticket = forgotten forever
```

| Check | PASS condition |
|---|---|
| All referenced stories linked | Every story mentioned in description/AC has a Jira issue link |
| Dependency sequence clear | If story B depends on story A, the dependency direction is explicit |
| Deferred items have tickets | Every "deferred to a later story" has a ticket number or is an explicit, accepted gap |
| Shared objects identified | If two stories modify the same object, both stories acknowledge it |

**FAIL examples (from real stories):**
- DATA-2525 → 2921 → 2924 → 2791: each references objects from the others. Pop B deferred, Prefect orchestration deferred, DQ checks deferred — all to "a later story" with no ticket number.
- Masking policies needing SYSADMIN, access grants needing a user list — cross-team dependencies invisible to the story.

### DIM-12: DECISION OWNERSHIP — Who resolves tradeoffs?

```
Does this story involve business logic where the "right" answer could go multiple ways?
  YES → Does the story name a decision-maker for business tradeoffs?
    YES → PASS
    NO  → FAIL — engineer will discover the tradeoff, draft a message, wait for approval, 
           lose a day. The story should pre-authorize or pre-decide.
  NO  → Straightforward build with no ambiguity? → PASS
```

| Check | PASS condition |
|---|---|
| Business decision-maker named | Story identifies who to escalate to if the AC seems disproportionate to the data reality, or if the engineer finds an edge case not covered |
| Known tradeoffs pre-decided | If the BSA knows there are multiple valid approaches, the story picks one — not "engineer's choice" on a business rule |
| AC proportionality acknowledged | If AC requires complex logic, story confirms the complexity is warranted (or says "if data shows this is <N cases, simplify to [APPROACH]") |

**FAIL examples (from real stories):**
- Full carry-forward logic required by AC for ENTERPRISEID reversions. Data showed 1 dealer out of 36K ever had one. Engineer had to escalate to Perry to get permission to simplify.
- Hash vs field-by-field delta — engineer proposed it, story forbade structural changes, engineer self-rejected. Then BSA didn't know the answer either, had to ask upstream.

---

## Output Format

### Section 1: Verdict Banner

```
╔══════════════════════════════════════════╗
║  VERDICT: READY FOR DEV  /  NOT READY   ║
║  Gaps: X critical, Y warning            ║
╚══════════════════════════════════════════╝
```

**READY FOR DEV** = zero critical gaps, ≤ 2 warnings (all with reasonable defaults an engineer could assume).
**NOT READY** = any critical gap exists.

### Section 2: Gaps Table

One row per gap found. Sorted: Critical first, then Warning.

| # | Dimension | Gap | Severity | What's Missing | Suggested Question for BSA |
|---|-----------|-----|----------|----------------|---------------------------|
| 1 | BIZ LOGIC | Business filter undefined | Critical | Story says "active contacts" but doesn't define which statuses count as active | "Which exact status values count as 'active'? List every valid value." |
| 2 | BIZ LOGIC | Business key not identified | Critical | Story doesn't say which field uniquely identifies a dealer from DC's perspective | "Is DCID the unique identifier per dealer, or is it LenderDealerID, or a combination?" |
| 3 | OPERATIONAL | Vendor SLA missing | Warning | No SLA documented for file delivery issues | "If the daily file is missing, what's DealerCenter's committed response time? Who do we email?" |

**Column rules:**
- **#**: Sequential, never renumbered.
- **Dimension**: One of DIM-1 through DIM-12.
- **Gap**: Short label — 10 words max.
- **Severity**: `Critical` = blocks dev. `Warning` = engineer could assume a default but shouldn't have to.
- **What's Missing**: One sentence. What the story says (or doesn't) and why it's a problem.
- **Suggested Question for BSA**: Copy-pasteable question. The engineer sends this verbatim to the BSA. No fluff, no "could you clarify" — just the question.

### Section 3: What's Good

2–3 bullet max. Only if genuinely well-specified areas exist. Not filler.

### Section 4: Estimability Assessment

```
ESTIMABILITY: CAN ESTIMATE / CANNOT ESTIMATE

Known scope (engineer can size these):
  - [list concrete deliverables the story defines clearly enough to estimate]

Unknown scope (blocks estimation):
  - [list items where the engineer can't guess effort because the story is silent]
```

Example:
```
ESTIMABILITY: CANNOT ESTIMATE

Known scope:
  - SFTP ingestion + decryption pipeline (standard pattern, ~2 pts)
  - Net-new Snowflake table creation (~1 pt)
  - Alerting on failure (~0.5 pt)

Unknown scope:
  - SCD2 vs snapshot (adds 2-5 pts depending on answer)
  - Business key definition (affects merge complexity)
  → Resolve gaps #1 and #2 before pointing this story
```

### Section 5: Rewrite Suggestions (if NOT READY)

For each critical gap, show what a **ready** version of that section would look like. Use placeholder values in `[BRACKETS]` where the BSA needs to fill in.

Example:
```
GAP #1 — Business filter undefined
CURRENT:  "Load active contacts into the DW."
READY:    "Load contacts where DELIVERYSTATUS is one of: 'Successful', 'Unsuccessful', 'Unsubscribed'. 
           Exclude contacts with ISDELETED = true in Salesforce."

GAP #2 — Vendor SLA missing
CURRENT:  (not mentioned)
READY:    "DealerCenter SLA: file delivered by 3 AM PST daily. If file is missing by 8 AM PST, 
           contact Rahul Sonthalia (rsonthalia@nowcom.com). DealerCenter has committed to a 
           1 business day response time for file delivery issues."
```

---

## Anti-Patterns — Never Do These

1. **Don't flag HOW as a gap.** Table naming, SCD design, retry logic, deployment scripts, staging patterns, error handling — these are engineering decisions. If you catch yourself writing a gap about implementation, stop. Run it through the BSA vs. Engineering boundary tree.
2. **Don't assume what the BSA meant.** If the story says "the contact table" and you know there's a `DIM_CONTACT` and a `FACT_CONTACT_EXPORT`, flag it — don't pick one.
3. **Don't pass a story because it's "mostly clear."** Mostly clear = ambiguous = not ready.
4. **Don't write novels in the gaps table.** Short labels, one-sentence explanations, copy-paste questions. That's it.
5. **Don't flag things as gaps that are team-standard defaults.** If the architecture standards skill defines a default behavior (e.g., IT fields are always added), that's not a gap — it's a known convention.
6. **Don't soften the verdict.** The story is ready or it isn't. No "mostly ready" or "ready with minor clarifications." If clarifications are needed, it's NOT READY.
7. **Don't suggest questions about engineering concerns.** The BSA questions column must only contain questions the BSA can actually answer — business rules, vendor info, SLAs, data definitions. Never ask a BSA "what Snowflake stage should we use?" or "should this be SCD2?"

---

## Interaction With Other Skills

| Situation | Skill / Tool to Use |
|---|---|
| Story references existing Snowflake objects (DIM-10) | **Query Snowflake via MCP** — verify object exists, column names match, data types match, nullable assumptions hold. This is not optional when MCP is available. |
| Story references Jira tickets as dependencies (DIM-11) | **Query Jira via Atlassian Rovo MCP** — verify linked stories exist and check their status. Flag if dependency is not Done or in same sprint. |
| Story includes SQL snippets | Load `sql-peer-reviewer-skill` for code review |
| Story references deployment scripts | Load `monitoring_standards.md` from project files for VALIDATE.sql pattern |
| Story references DeFi custom fields | Load `defi-field-coder-skill` for pipeline completeness |

---

## ADHD-Friendly Output Rules (Always Applied)

Borrowed from the `i-have-adhd` skill — these shape every response from this skill:

1. **Verdict first.** The banner is line 1. Not context, not a summary of the story, not "let me review this."
2. **Table before prose.** Gaps table is the second thing. Detailed explanations come after, if at all.
3. **Estimability right after gaps.** The engineer needs to know: can I point this or not?
4. **Cap the gaps table at the 5 most critical.** If there are more, add a line: "Plus N additional warnings — fixing the critical gaps first will likely resolve several of these."
5. **Copy-paste ready.** The BSA questions column is designed to be pasted directly into a Jira comment or Slack message. No rewording needed.
6. **No preamble, no recap, no closing pleasantries.** Start with the verdict. End when the analysis is done.
7. **One next action at the end.** Either "Send gaps #1–#3 to BSA for answers, then point" or "Story is ready — point and assign to sprint."

---

## HTML Report (Optional — On Request)

If the user asks for an artifact or report, generate a self-contained HTML report matching the style of the SQL peer review report:
- Verdict banner (green = READY, red = NOT READY)
- Gaps table with severity color-coding (Critical = red, Warning = amber)
- Clean, print-friendly, no external dependencies
- Footer: "Generated by Claude Story Validator — GLS Auto DW Team"

Only generate when explicitly asked — default output is inline markdown.
