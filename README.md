# dw-engineer

Claude Code plugin for the GLS Auto Data Warehouse & BI Team. Gives Claude deep knowledge of GLS-specific standards, peer review frameworks, and code optimization guidance for Snowflake SQL, Python/Prefect pipelines, and reporting objects.

---

## Skills

| Skill | Trigger phrases | What it does |
|---|---|---|
| **sql-peer-reviewer** | "review this SQL", "peer review", "check my code", "is this SQL correct" | 6-area SQL peer review: Correctness, Completeness, Performance, Maintainability, Best Practices, Security. Produces a formatted + corrected query. Auto-invokes `sql-formatter` and `code-optimizer`. |
| **python-peer-reviewer** | "review this Python", "peer review", "check my flow", "is this Prefect code correct" | 7-area Python/Prefect peer review against GLS coding standards, Prefect naming conventions, and pipeline best practices. |
| **snowflake-sql-architecture-standards** | "does this follow standards", "what should I name this", "how should I structure this", paste any Snowflake DDL | GLS Snowflake architecture reference: naming conventions, table/sproc/view DDL templates, Kimball modeling decision tree, IT fields, DQ framework, reporting standards. |
| **code-optimizer** | "optimize this", "make this faster", "this is slow", "performance issues" | Two-track optimization: Quick Wins (drop-in fixes) + Full Refactor (best possible version). Covers Snowflake SQL and Python. |
| **sql-formatter** | "format this SQL", "clean up this query" — also auto-invoked by `sql-peer-reviewer` | Reformats Snowflake SQL to DW Standards — casing, indentation, clause layout, JOINs, CTEs, semicolons. Returns formatted SQL only, no commentary. |

### Skill dependencies

`sql-peer-reviewer` automatically invokes two other skills during its review:
- **`sql-formatter`** — produces the Formatted Query section at the end of every review
- **`code-optimizer`** — drives the Performance area analysis

All five skills are bundled together for a complete SQL peer review workflow.

---

## Installation

### From the Plugin Directory (recommended)

1. Open Claude Code
2. Open **Directory** > **Plugins**
3. Find **dw-engineer** under the **Code** tab
4. Click to install

### From settings.json

Add to `~/.claude/settings.json`:

```json
{
  "plugins": [
    { "path": "/path/to/dw-engineer-plugin" }
  ]
}
```

---

## Requirements

- Claude Code (desktop app or CLI)
- No external dependencies — all skills are knowledge-only (no MCP servers, no hooks)

---

## Contributing

1. Clone the repo
2. Edit or add `SKILL.md` files under `skills/<skill-name>/`
3. Open a pull request
