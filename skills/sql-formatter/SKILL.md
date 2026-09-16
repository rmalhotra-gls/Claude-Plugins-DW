---
name: sql-formatter
description: >
  Formats Snowflake SQL code according to DW Standards. Always trigger this
  skill when the user asks to format, clean up, prettify, or standardize SQL code.
  Also trigger when another skill (such as sql-peer-reviewer) requests a formatted
  version of a query as part of its output. Never skip formatting when explicitly
  invoked — always return a fully reformatted query block with no commentary, only
  the formatted SQL.
---

# SQL Formatter — DW Standards

## Purpose

Reformat Snowflake SQL to conform to DW Standards. The output is always
**formatted SQL only** — no explanations, no commentary, no preamble. Just the code.

---

## What the Formatter Changes

Only two categories of changes are ever made:

1. **Casing** — SQL keywords, built-in function names, and all identifiers (column
   names, table names, CTE names, aliases) are uppercased.
2. **Indentation and spacing structure** — clause layout, tab indentation, operator
   spacing, and comma placement are normalized per the rules below.

Nothing else changes. Ever.

---

## What the Formatter Never Touches

- Anything inside single quotes — `'Blue'` stays `'Blue'`, `'AUTO'` stays `'AUTO'`,
  `'true'` stays `'true'`. Values being compared against are never modified.
- Anything inside double quotes — `"Loan Amount"` stays `"Loan Amount"` exactly.
- Comments — preserved verbatim, not moved or reformatted.
- Logic, statement order, or structure — nothing is rewritten, only reformatted.

---

## Formatting Rules

### 1. Casing

Everything not inside quotes is uppercased — keywords, function calls, identifiers,
aliases, all of it. Single quotes and double quotes are the only shield: anything
inside them is left completely untouched.

---

### 2. Indentation

Use **one tab** per indentation level. Never use spaces for indentation.

---

### 3. Clause Layout

Each major clause sits at **zero indentation on its own line**. Everything that
belongs to that clause goes on the following line(s), indented one tab:

```sql
SELECT
	COL_A,
	COL_B
FROM
	SOME_TABLE
WHERE
	COL_A = 'value'
```

Applies to: `SELECT`, `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `QUALIFY`,
`LIMIT`, `CASE`, `ELSE`, `END`.

---

### 4. Commas

Commas are always **trailing** — at the end of the line, never at the start.
Always place a single space after every comma:

```sql
SELECT
	COL_A,
	COL_B,
	COL_C
```

---

### 5. Operators

Always place a single space before and after mathematical and comparison operators:

```sql
-- ✅ Correct
LOAN_AMT + FEE_AMT
STATUS = 'Funded'
AMOUNT > 0
RATE * 100

-- ❌ Wrong
LOAN_AMT+FEE_AMT
STATUS='Funded'
```

---

### 6. Spacing

Tokens are separated by a single space only. No mass alignment, no padding to
align across multiple lines:

```sql
-- ✅ Correct
LOAN_ID AS LOANID,
LOAN_AMT AS LOANAMT,

-- ❌ Wrong — padded to align
LOAN_ID  AS LOANID,
LOAN_AMT AS LOANAMT,
```

---

### 7. JOINs

The JOIN keyword and the joined table name sit on the same line at zero indentation,
directly beneath the base table. `ON` and its conditions are each on their own line,
indented one tab. Additional `AND` / `OR` conditions within `ON` are also indented
one tab:

```sql
FROM
	LOANS L
	INNER JOIN CUSTOM_FIELDS CF
		ON L.LOAN_ID = CF.LOAN_ID
		AND CF.IS_ACTIVE = 'true'
```

---

### 8. CASE Expressions

`CASE` sits at one tab (as it is a column expression inside `SELECT`). `WHEN` is
indented one additional tab. `THEN` is indented one additional tab under `WHEN`.
`ELSE` sits at the same level as `WHEN`. `END` and its alias sit at the same level
as `CASE`:

```sql
SELECT
	CASE
		WHEN LOAN_TYPE = 'AUTO' AND LOAN_AMT > 0
			THEN 'Eligible'
		ELSE NULL
	END AS ELIGIBILITYFLAG
```

---

### 9. CTEs

`WITH` sits at zero indentation. Each CTE name and `AS` sit on the same line at zero
indentation. The opening `(` drops to the next line at zero indentation. The CTE body
is indented one tab. The closing `),` sits at zero indentation and runs directly into
the next CTE name with no blank line between them. The final `SELECT` follows
immediately after the last CTE's closing `)` with no blank line:

```sql
WITH FIRSTCTE AS
(
	SELECT
		COL_A,
		COL_B
	FROM
		SOME_TABLE
),
SECONDCTE AS
(
	SELECT
		COL_X,
		COL_Y
	FROM
		ANOTHER_TABLE
)
SELECT
	F.COL_A,
	S.COL_X
FROM
	FIRSTCTE F
	INNER JOIN SECONDCTE S
		ON F.COL_A = S.COL_X
```

---

### 10. Semicolons

The semicolon closes the statement on the last line of the final clause, with no
space before it:

```sql
WHERE
	STATUS = 'Funded';
```

---

## Reference Example

```sql
SELECT
	LOANID AS LOANID,
	LOANAMT AS "Loan Amount",
	CASE
		WHEN LOANTYPE = 'AUTO' AND LOANAMT > 0
			THEN 'Eligible'
		ELSE NULL
	END AS ELIGIBILITYFLAG
FROM
	LOANS L
	INNER JOIN CUSTOMFIELDS CF
		ON L.LOANID = CF.LOANID
		AND CF.ISACTIVE = 'true'
WHERE
	L.STATUS = 'Funded'
QUALIFY
	ROW_NUMBER() OVER (PARTITION BY LOANID ORDER BY IT_INSERTDATE DESC) = 1;
```

Key things to note in this example: `LOANID`, `LOANAMT`, `CUSTOMFIELDS`, and all other
unquoted identifiers are uppercased exactly as written — no underscores added, no
characters changed, only case applied. `"Loan Amount"` is left exactly as written
because it is inside double quotes. `'AUTO'`, `'Eligible'`, `'true'`, and `'Funded'`
are left exactly as written because they are inside single quotes.

---

## Output Contract

Return only the formatted SQL block. No preamble, no explanation, no trailing
commentary. If the input contains multiple statements, format each one and separate
them with a single blank line. Preserve statement order exactly as given.
