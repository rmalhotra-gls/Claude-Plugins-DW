# Claude Code Plugins — Quick Reference

---

## What is a plugin?

A plugin is a folder of instructions that teaches Claude how your team works. When you install a plugin, Claude automatically follows those instructions every time they apply — you don't have to repeat yourself.

Think of it like this: instead of telling Claude "here are our SQL formatting rules" every single time, you write those rules once in a plugin, and Claude knows them forever.

---

## 1. How We Built Our Plugin

We put everything in a GitHub repo. The repo has a specific folder layout that Claude Code knows how to read. That's it — no special tools, no compiling, no building. Just folders and text files.

### What the folder looks like

```
dw-engineer-plugin/
│
├── .claude-plugin/
│   └── marketplace.json        ← tells Claude "here's a plugin in this repo"
│
├── plugins/
│   └── dw-engineer/
│       ├── .claude-plugin/
│       │   └── plugin.json     ← name, version, who made it
│       └── skills/
│           ├── sql-peer-reviewer/
│           │   └── SKILL.md    ← one skill = one file of instructions
│           ├── sql-formatter/
│           │   └── SKILL.md
│           ├── snowflake-sql-architecture-standards/
│           │   └── SKILL.md
│           ├── python-peer-reviewer/
│           │   └── SKILL.md
│           └── code-optimizer/
│               └── SKILL.md
│
├── .gitignore
└── README.md
```

### How to build one from scratch (step by step)

**Step 1 — Make the folders**

Create the folder structure shown above. Every plugin needs this exact layout. The names can change, but the structure can't.

**Step 2 — Create `marketplace.json`**

This file lives in `.claude-plugin/marketplace.json` at the top of the repo. It tells Claude Code "this repo contains a plugin, and here's where to find it."

```json
{
  "name": "your-plugin-name",
  "owner": { "name": "Your Team Name" },
  "metadata": {
    "description": "One sentence about what this plugin does.",
    "version": "1.0.0"
  },
  "plugins": [
    {
      "name": "your-plugin-name",
      "description": "A longer description of what skills are included.",
      "version": "0.1.0",
      "source": "./plugins/your-plugin-name",
      "author": { "name": "Your Team Name" },
      "homepage": "https://github.com/your-org/your-repo",
      "category": "development"
    }
  ]
}
```

**Step 3 — Create `plugin.json`**

This file lives inside the plugin folder at `plugins/your-plugin-name/.claude-plugin/plugin.json`. It's the plugin's ID card.

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

**Step 4 — Write your skills**

Each skill is a separate folder inside `plugins/your-plugin-name/skills/`. Each folder has one file: `SKILL.md`.

Every `SKILL.md` file starts with a header block (called "frontmatter") that tells Claude when to use it:

```markdown
---
name: skill-name
description: "Tell Claude when to use this skill. Be very specific."
---

# Skill Title

Write the actual instructions here. This is what Claude reads
and follows when the skill gets triggered.
```

**Step 5 — Add a README.md**

Put this at the top of the repo. List what skills are included and how to install. This is for humans, not Claude.

**Step 6 — Add a .gitignore**

Keeps local junk out of the shared repo:

```
.claude/*.local.md
.claude/settings.local.json
.DS_Store
Thumbs.db
.vscode/
.idea/
```

**Step 7 — Push it to GitHub**

Put the whole thing in a GitHub repo. Once it's there, anyone with access can install it.

```bash
git init
git add -A
git commit -m "Initial commit"
git remote add origin https://github.com/your-org/your-repo.git
git push -u origin main
```

**Step 8 — Done**

When you update a skill, push to GitHub. Everyone who has the plugin gets the update next time they open Claude Code. No reinstalling, no downloading, no extra steps.

---

## 2. When to Use a Plugin

**Use a plugin when the same instructions need to be shared across multiple people.**

| Situation | Use a plugin? | Why |
|---|---|---|
| The whole team needs to follow the same SQL formatting rules | Yes | Everyone gets the same rules, no one drifts |
| Code reviews need to check the same things every time | Yes | Consistency — Claude doesn't forget a checklist item |
| Multiple skills depend on each other (reviewer needs the formatter) | Yes | They all live together in one package |
| A new person joins and needs Claude to already know your standards | Yes | No onboarding doc needed — Claude just knows |
| Your standards change and everyone needs the update at once | Yes | You update once, push, everyone gets it |
| You want a record of what instructions Claude was given | Yes | It's all in Git — you can see who changed what and when |

**In plain English:** if more than one person needs Claude to do the same thing the same way, make it a plugin.

---

## 3. When NOT to Use a Plugin

**Don't use a plugin for things that are just for you, or just for one project.**

| Situation | What to do instead |
|---|---|
| A personal preference (like "always use dark mode examples") | Put it in your personal Claude settings file |
| Instructions that only matter for one specific project | Put a `CLAUDE.md` file in that project's folder |
| A one-time instruction ("summarize this meeting") | Just tell Claude in the chat — no file needed |
| Passwords, API keys, or anything secret | **Never** put secrets in a shared plugin |
| Something you're just trying out | Test it locally first, make it a plugin later if it works |

**In plain English:** if it's just you, or just this one time, or it contains anything sensitive — skip the plugin.

---

## 4. Team Setup — How to Install the Plugin

### Easiest way (from GitHub URL)

If the repo is accessible to you on GitHub:

1. Open **Claude Code** (the desktop app or CLI)
2. Go to **Plugins**
3. Choose **Add from URL** (or "Install from GitHub")
4. Paste this URL:
   ```
   https://github.com/rmalhotra-gls/Claude-Plugins-DW
   ```
5. Confirm the install
6. Done — the 5 skills are now active

### Alternative way (manual clone)

If the marketplace install doesn't work (private repo, network issues, etc.):

**Step 1 — Download the repo**

Open a terminal and run:

```bash
git clone https://github.com/rmalhotra-gls/Claude-Plugins-DW.git
```

Save it somewhere you won't accidentally delete it.
Good spot: `C:\Users\YourName\Documents\Claude-Plugins-DW`

**Step 2 — Tell Claude Code where you saved it**

Find (or create) this file on your computer:
`C:\Users\YourName\.claude\settings.json`

Open it in any text editor and add this:

```json
{
  "plugins": [
    { "path": "C:\\Users\\YourName\\Documents\\Claude-Plugins-DW" }
  ]
}
```

Replace `YourName` with your actual Windows username. The double backslashes (`\\`) are required on Windows.

**Step 3 — Check that it works**

Open Claude Code and type something like:

> review this SQL

If Claude responds with a structured 6-area peer review, the plugin is working.

### How to get updates

**If you installed from GitHub URL:** updates happen automatically when the repo is updated. Nothing to do.

**If you installed manually:** open a terminal, go to where you saved the repo, and run:

```bash
git pull
```

The next time you open Claude Code, it picks up the changes.

### Something not working?

| What's happening | What to check |
|---|---|
| Claude doesn't use the skills | Make sure the path in `settings.json` is correct and uses double backslashes |
| "Plugin not found" error | Make sure the folder you pointed to has a `.claude-plugin` folder inside it |
| Getting an old version of a skill | Run `git pull` in the plugin folder to get the latest |
| Can't find `settings.json` | Create it yourself at `C:\Users\YourName\.claude\settings.json` |
| Can't clone the repo | Ask for access to the GitHub repo from the DW team |
