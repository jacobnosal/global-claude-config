# global-claude-config

Global Claude Code configuration — applies to every project on every machine.

---

## What's in This Repo

```
global-claude-config/
├── CLAUDE.md           Global instructions: identity, hard rules, session habits, backlog workflow
├── settings.json       Auto-approved tools (git, gh, podman, npm, pytest, ruff, bandit, etc.)
└── skills/             Slash commands available in every Claude Code session
    ├── daily/          /daily          — Daily planning from all BACKLOG.md files
    ├── backlog-add/    /backlog-add    — Add item to a project BACKLOG.md
    ├── backlog-done/   /backlog-done   — Mark backlog item complete
    ├── gh-plan/        /gh-plan        — Complexity assessment + issue hierarchy
    ├── gh-issue/       /gh-issue       — Create single GitHub issue
    ├── gh-start/       /gh-start       — Start work on an issue
    ├── gh-finish/      /gh-finish      — Quality gates + DoD + open PR
    ├── gh-review/      /gh-review      — Address PR review feedback
    └── git-sync/       /git-sync       — Rebase on develop + run quality gates
```

## How It Works

Claude Code reads `~/.claude/` for global config. On this machine, those files are
symlinked here so edits are automatically tracked in git:

```
~/.claude/CLAUDE.md      → global-claude-config/CLAUDE.md
~/.claude/settings.json  → global-claude-config/settings.json
~/.claude/skills/        → global-claude-config/skills/
```

Any skill you add or edit is immediately live in Claude Code and tracked in git.

---

## New Machine Setup

### 1. Prerequisites

```bash
brew install git node gh podman podman-compose python
npm install -g @anthropic/claude-code
gh auth login
```

### 2. Clone this repo

```bash
git clone https://github.com/jacobnosal/global-claude-config.git ~/Library/CloudStorage/OneDrive-Accenture/projects/global-claude-config
```

> If OneDrive isn't synced yet, clone anywhere and adjust the symlink paths below.

### 3. Create symlinks

```bash
# Back up existing ~/.claude config if present
mv ~/.claude/CLAUDE.md ~/.claude/CLAUDE.md.backup 2>/dev/null || true
mv ~/.claude/settings.json ~/.claude/settings.json.backup 2>/dev/null || true
mv ~/.claude/skills ~/.claude/skills.backup 2>/dev/null || true

REPO=~/Library/CloudStorage/OneDrive-Accenture/projects/global-claude-config

ln -s "$REPO/CLAUDE.md"      ~/.claude/CLAUDE.md
ln -s "$REPO/settings.json"  ~/.claude/settings.json
ln -s "$REPO/skills"         ~/.claude/skills
```

### 4. Verify

Open a Claude Code session and type `/` — all skills should appear in the menu.

---

## Adding a New Skill

```bash
mkdir -p skills/<skill-name>
# Create skills/<skill-name>/SKILL.md with YAML frontmatter
# Immediately live in Claude Code — no restart needed
git add -A && git commit -m "feat(skills): add /<skill-name>"
git push
```

## Pushing Changes

```bash
git add -A
git commit -m "chore(claude): <description>"
git push
```

## Pulling Updates on Another Machine

```bash
git pull
# Changes are live immediately via symlinks
```
