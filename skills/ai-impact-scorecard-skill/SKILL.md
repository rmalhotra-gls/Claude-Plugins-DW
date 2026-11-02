---
name: ai-impact-scorecard-skill
description: Measures every AI project before vs after on four dimensions — Dollars, Hours, Accuracy, Adoption — using a fixed question set and a PRE/POST check format. Use whenever the user wants to capture a baseline before an AI project goes live, measure results after go-live, build an ROI or impact scorecard, justify an AI initiative, compare manual vs AI process, or answer "was this AI project worth it". Always trigger on "AI impact", "AI ROI", "before and after", "baseline", "scorecard", "hours saved", "dollars saved", "adoption", "accuracy of the AI", "30/60/90 day review", or /ai-impact-scorecard-skill.
---

# AI Impact Scorecard: before vs after

One scorecard per AI project. Two capture points:

- **PRE** — baseline of the process as it runs today, captured BEFORE go-live.
- **POST** — same measurements, same method, at checkpoints after go-live (default: 30 and 90 days).

The scorecard uses the same output schema as the team's MONITORING/VALIDATE scripts, so anyone can read it without asking the author.

---

## Hard rules (override everything)

1. **Never invent a number.** Every value has a source: a query, a Jira/Confluence link, a log, an invoice, or "owner-stated" with the person's role. No source → the check is `FAIL` with `ACTUAL = 'no sourced value'`.
2. **Never invent a rate.** Loaded hourly rate, license price and credit price come from the user. Never assume one.
3. **Same method PRE and POST.** Same unit of work, same period length, same definition of "error", same sampling method. If POST used a different method, the check is `FAIL` and ACTUAL names the difference.
4. **Hours saved ≠ dollars saved.** Hours become dollars only if the user confirms the time was redeployed, a cost was removed, or a hire was avoided. Otherwise report them as **capacity freed**, not savings.
5. **AI time includes human time.** POST hours always include review, correction and exception handling of AI output. A POST hours value without review time is `FAIL`.
6. **A baseline captured after go-live is labeled `RETRO`.** It is allowed, but every check that depends on it carries `(RETRO baseline)` in ACTUAL.
7. **No customer PII** anywhere in the scorecard. Use counts, IDs or ticket keys.

---

## Step 0: Pull before asking

Before asking any question, try to answer it from tools. Ask the user only for what tools cannot return.

| Data | Try first |
|---|---|
| Project scope, owner, go-live date, linked stories | Jira epic (Atlassian MCP) |
| Process description, SOPs | Confluence |
| Volume of work units | Snowflake query on the process's own tables / logs |
| Compute and AI run cost | Snowflake usage views (credits, Cortex usage) |
| Time spent | Jira worklogs, if the team logs time |
| Rework / defects | Jira bugs, rework tickets, PR comments linked to the process |

Record every pulled value with its source. Then ask only the remaining questions.

---

## Step 1: Question set

Ask in batches, one dimension at a time. Questions are yes/no or short answer. Skip any question Step 0 already answered.

### S — Scope (ask once, at PRE)

| # | Question | Answer type |
|---|---|---|
| S1 | Project name and Jira epic key? | short |
| S2 | What process does the AI replace or assist? | one line |
| S3 | What is one **unit of work**? (one ticket, one review, one report, one record) | short |
| S4 | Who does the work today? (role, # of people) | short |
| S5 | Planned go-live date? | date |
| S6 | Is go-live already past? | yes/no → if yes, baseline = `RETRO` |
| S7 | Measurement period length? (default 30 days) | short |

### D — Dollars

| # | Phase | Question | Answer type |
|---|---|---|---|
| D1 | PRE | Loaded hourly rate per role doing the work? | $ / hr |
| D2 | PRE | Recurring tool / license / vendor cost for this process today? | $ / month, or 0 |
| D3 | PRE | Cost of errors today (rework, credits, penalties)? Known or unknown? | $ / month, or "unknown" |
| D4 | POST | One-time build cost: engineer hours spent building it? | hours |
| D5 | POST | Recurring AI run cost? (Snowflake credits, Cortex, API, seats) | $ / month |
| D6 | POST | Maintenance hours per month (prompt fixes, re-runs, support)? | hours / month |
| D7 | POST | Were freed hours redeployed, a cost removed, or a hire avoided? | yes/no + which |

**Derived:**
- PRE monthly cost = (PRE hours/month × D1) + D2 + D3
- POST monthly cost = (POST hours/month × D1) + D5 + (D6 × D1) + POST error cost
- Monthly net = PRE monthly cost − POST monthly cost
- Payback months = (D4 × D1) ÷ monthly net (only if monthly net > 0)
- If D7 = no → net is reported as **capacity freed ($ equivalent)**, not savings.

### H — Hours

| # | Phase | Question | Answer type |
|---|---|---|---|
| H1 | PRE + POST | Units of work per period? | count |
| H2 | PRE + POST | Hands-on minutes per unit? Median, and how measured? (timed sample, worklog, estimate) | minutes + method |
| H3 | PRE + POST | Sample size behind H2? | count |
| H4 | PRE + POST | Elapsed (cycle) time per unit, request to done? | hours/days |
| H5 | POST | Human review minutes per AI output? | minutes |
| H6 | POST | % of units that fall back to the manual path (exceptions)? | % |

**Derived:**
- Hours/period = H1 × (H2 + H5) ÷ 60, plus exception units at PRE minutes/unit
- Hands-on change % and cycle-time change % reported separately. Faster cycle time with the same hands-on time is not an hours saving.

### A — Accuracy

| # | Phase | Question | Answer type |
|---|---|---|---|
| A1 | PRE | What counts as an error? (one-line definition, fixed for PRE and POST) | one line |
| A2 | PRE + POST | How is correctness checked? (ground truth, second reviewer, downstream defect) | short |
| A3 | PRE + POST | Sample size checked? | count |
| A4 | PRE + POST | Errors found in that sample? | count |
| A5 | POST | % of AI outputs a human overrode or edited materially? | % |
| A6 | POST | Any error that reached production or a customer? | yes/no + ticket keys |

**Derived:**
- Error rate = A4 ÷ A3, PRE vs POST
- If A3 < 30 at either phase → ACTUAL carries `(small sample, n=__)`

### U — Adoption

| # | Phase | Question | Answer type |
|---|---|---|---|
| U1 | POST | Eligible users? (people who should use it) | count |
| U2 | POST | Active users in period? (used it ≥1 time) | count |
| U3 | POST | % of eligible units of work that went through the AI path? | % |
| U4 | POST | Users active at 30 days still active at 90 days? | count |
| U5 | POST | Where is usage measured? (log table, Jira label, telemetry, survey) | short |
| U6 | POST | Top reason for not using it, if known? | one line or "unknown" |

**Derived:**
- User adoption % = U2 ÷ U1
- Work adoption % = U3 (this is the one that drives Dollars and Hours)
- Retention % = U4 ÷ users active at 30 days

---

## Step 2: Targets

At PRE, ask for one target per dimension. If the user gives none, the POST check reports the delta only and `RESULT = FAIL` with ACTUAL `'no target set'`. Do not suggest a target unless asked.

| Dimension | Target asked as |
|---|---|
| Dollars | Monthly net ≥ $__ , payback ≤ __ months |
| Hours | Hands-on minutes per unit ≤ __ (incl. review) |
| Accuracy | Error rate ≤ __ % (must be ≤ PRE unless user says otherwise) |
| Adoption | Work adoption ≥ __ % by day 90 |

---

## Step 3: Scorecard output

Same 4-column schema as VALIDATE scripts. One row per check.

| Column | Rule |
|---|---|
| `CHECKNAME` | `PRE_##_LABEL` or `POST_##_LABEL` (POST checks carry the checkpoint: `POST30_`, `POST90_`) |
| `RESULT` | `PASS` or `FAIL` only. No detail here. |
| `EXPECTED` | PRE: "sourced baseline value". POST: the target. |
| `ACTUAL` | Value, unit, source. POST rows always show **both sides and the delta**: `'18 min/unit (PRE 42, −57%) incl. 6 min review · src: worklog n=25'` |

PRE checks PASS when the value is captured and sourced. They FAIL when missing, unsourced, or measured with a method that cannot be repeated at POST.

### Example

```
AI Impact Scorecard — DATA-1234 SQL Peer Review Bot — POST30 — 2 FAIL

| CHECKNAME                  | RESULT | EXPECTED                    | ACTUAL                                                        |
|----------------------------|--------|-----------------------------|---------------------------------------------------------------|
| PRE_01_HOURS_PER_UNIT      | PASS   | Sourced baseline            | 42 min/PR (median, n=20) · src: timed sample, owner-stated    |
| PRE_02_ERROR_RATE          | PASS   | Sourced baseline            | 9.0% (9/100 PRs w/ prod defect) · src: Jira bugs DATA-*       |
| PRE_03_RATE                | FAIL   | Sourced loaded hourly rate  | no sourced value                                              |
| POST30_01_HOURS_PER_UNIT   | PASS   | ≤ 20 min/PR incl. review    | 18 min/PR (PRE 42, −57%) incl. 6 min review · n=25            |
| POST30_02_ERROR_RATE       | PASS   | ≤ 9.0%                      | 6.0% (6/100) (PRE 9.0%, −3.0 pts)                             |
| POST30_03_WORK_ADOPTION    | FAIL   | ≥ 70% of PRs via AI path    | 41% (82/200 PRs) · 118 manual · top reason: "slow on big PRs" |
| POST30_04_MONTHLY_NET      | PASS   | Capacity freed reported     | 80 hrs/month capacity freed · $ not computed (PRE_03 FAIL)    |
```

Lead with the header line: project, checkpoint, number of FAILs. Then the table. Then, only if there are FAILs, one line per FAIL saying what is needed to close it. Nothing else.

---

## Anti-patterns

- `'FAIL'` or `'improved'` alone in ACTUAL.
- POST value without the PRE value next to it.
- Hours saved converted to dollars without D7 confirmed.
- POST minutes that leave out human review.
- Changing the error definition between PRE and POST.
- High user adoption reported while work adoption is low. Report both.
- Accuracy from a sample the AI chose.
