# Claude Code Plugins — Quick Reference

---

## 1. How to Build a Plugin (Our Way)

One repo. Skills inside. Git-distributed. No build step.

### Folder Structure

```
your-plugin-name/
├── .claude-plugin/
│   └── marketplace.json        ← declares what plugins live in this repo
├── plugins/
│   └── your-plugin-name/
│       ├── .claude-plugin/
│       │   └── plugin.json     ← name, version, author
│       └── skills/
│           ├── skill-one/
│           │   └── SKILL.md    ← the actual skill instructions
│           ├── skill-two/
│           │   └── SKILL.md
│           └── skill-three/
│               └── SKILL.md
├── .gitignore
└── README.md
```

### Steps

**1 — Create the directories**

```
mkdir your-plugin-name
cd your-plugin-name
mkdir .claude-plugin
mkdir -p plugins/your-plugin-name/.claude-plugin
mkdir -p plugins/your-plugin-name/skills
```

**2 — Write `marketplace.json`** (root level)

Goes in `.claude-plugin/marketplace.json`. This is the repo-level manifest.

```json
{
  "name": "your-plugin-name",
  "owner": { "name": "Your Team Name" },
  "metadata": {
    "description": "One sentence. What this plugin gives Claude.",
    "version": "1.0.0"
  },
  "plugins": [
    {
      "name": "your-plugin-name",
      "description": "Longer description of what skills are bundled.",
      "version": "0.1.0",
      "source": "./plugins/your-plugin-name",
      "author": { "name": "Your Team Name" },
      "homepage": "https://github.com/your-org/your-repo",
      "category": "development"
    }
  ]
}
```

**3 — Write `plugin.json`** (inside the plugin)

Goes in `plugins/your-plugin-name/.claude-plugin/plugin.json`.

```json
{
  "name": "your-plugin-name",
  "version": "0.1.0",
  "description": "Same one-liner as above.",
  "author": {
    "name": "Your Team Name",
    "email": "you@company.com"
  }
}
```

**4 — Add skills**

One folder per skill under `plugins/your-plugin-name/skills/`.
Each folder contains exactly one file: `SKILL.md`.

Every `SKILL.md` starts with frontmatter:

```markdown
---
name: skill-name
description: "When to trigger this skill. Be specific — list exact phrases."
---

# Skill Title

[The actual instructions Claude follows when this skill fires.]
```

**5 — Add README.md**

At repo root. Table of skills, trigger phrases, install instructions.

**6 — Add .gitignore**

```
.claude/*.local.md
.claude/settings.local.json
.DS_Store
Thumbs.db
.vscode/
.idea/
```

**7 — Git init and push**

```bash
git init
git add -A
git commit -m "Initial commit"
git remote add origin https://github.com/your-org/your-repo.git
git push -u origin main
```

**8 — Team installs**

Each team member adds to their `.claude/settings.json`:

```json
{
  "plugins": [
    { "path": "/path/to/cloned/repo" }
  ]
}
```

Or per-session: `cc --plugin-dir /path/to/repo`

**9 — Updates**

You push to GitHub → team pulls → skills update automatically. No rebuild. No reinstall.

---

## 2. When to Use a Plugin

| Situation | Plugin? | Why |
|---|---|---|
| Team shares the same coding standards | **Yes** | One source of truth, version-controlled |
| Peer review needs to follow a specific framework every time | **Yes** | Consistency across reviewers |
| Multiple skills need to work together (reviewer calls formatter) | **Yes** | Bundle dependencies in one package |
| New team members need Claude to "just know" your patterns | **Yes** | Onboarding without tribal knowledge |
| Standards change and everyone needs the update | **Yes** | Push once, team pulls, done |
| You want governance over what Claude knows | **Yes** | Git history, PRs, code review on the skills themselves |
| Formatting rules that must be identical across the team | **Yes** | No drift between individual setups |

### The short version

Use a plugin when:
- **Multiple people** need the **same instructions**
- Those instructions **change over time** and everyone needs the latest
- You want **git history** on what Claude was told to do

---

## 3. When NOT to Use a Plugin

| Situation | Better alternative |
|---|---|
| One-off personal preference ("I like tabs over spaces") | User-level `CLAUDE.md` or `settings.json` |
| Project-specific context that only applies to one repo | Project `CLAUDE.md` in the repo root |
| Temporary instructions for a single task | Just tell Claude in chat |
| MCP server config that varies per machine | `.mcp.json` at project level |
| Something only you care about | `~/.claude/CLAUDE.md` (user-level) |
| Quick experiment or prototype skill | Single `SKILL.md` in `.claude/skills/` locally |
| Instructions that reference secrets or local paths | Never put these in a shared plugin |

### The short version

Skip the plugin when:
- It's **just you** — use personal config files instead
- It's **just this repo** — use project-level CLAUDE.md
- It's **throwaway** — just say it in chat
- It contains **secrets or machine-specific paths** — never distribute those

---

## 4. Team Setup — Getting Everyone on the Plugin

### What each person needs

- Claude Code CLI installed
- Git access to the plugin repo

### Steps (5 minutes)

**1 — Clone the repo**

```bash
git clone https://github.com/rmalhotra-gls/Claude-Plugins-DW.git
```

Pick a stable location. Don't put it inside another project.
Suggested: `C:\Users\<you>\Documents\Claude-Plugins-DW`

**2 — Tell Claude Code where it is**

Open (or create) `~/.claude/settings.json` and add:

```json
{
  "plugins": [
    { "path": "C:\\Users\\<you>\\Documents\\Claude-Plugins-DW" }
  ]
}
```

Replace `<you>` with your Windows username.

**3 — Verify it loaded**

Open Claude Code and type:

```
review this SQL
```

If Claude responds with the 6-area peer review format, it's working.

**4 — Staying up to date**

When skills get updated, just pull:

```bash
cd C:\Users\<you>\Documents\Claude-Plugins-DW
git pull
```

Next Claude Code session picks up the changes automatically. No reinstall.

### Troubleshooting

| Problem | Fix |
|---|---|
| Skills don't trigger | Check the path in `settings.json` — backslashes must be doubled (`\\`) on Windows |
| "Plugin not found" | Make sure the cloned folder has `.claude-plugin/marketplace.json` at root |
| Old version of a skill | Run `git pull` in the plugin folder |
| `settings.json` doesn't exist | Create it at `~/.claude/settings.json` (that's `C:\Users\<you>\.claude\settings.json` on Windows) |
