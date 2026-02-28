# Global Instructions

## User Profile

- **Name**: Jacob Nosal
- **Role**: CAS (Cloud & Application Security) Financial Services Practice Lead, Accenture Security
- **Container runtime**: Always Podman and podman-compose. Never Docker or docker-compose.
- **Enterprise Claude plan**: MCP server access disabled by org policy.

## Global Hard Rules

These apply in every session across every project, without exception.

| # | Rule |
|---|------|
| 1 | Do not write code before confirming scope — summarize and wait for confirmation |
| 2 | Do not push if any quality gate fails — stop and report; never suppress |
| 3 | Do not modify files outside the stated scope — ask first |
| 4 | Do not merge your own PRs — open them for user review |
| 5 | Do not deviate from project SKILL.md patterns silently — flag and get approval |
| 6 | Do not suppress security findings — report Medium/High before pushing |
| 7 | Report mistakes on committed code immediately — do not hide or silently fix |
| 8 | Do not refactor or improve code outside the issue scope |

## Session Habits

- Name every session immediately: `/rename <context-slug>` (e.g., `GH-42-auth`, `planning-sprint-4`)
- Run startup ritual at the start of every session: `git status`, `gh pr list --author "@me"`
- Output a Session Summary at the end of every work session (completed / in-progress / blocked / next session recommendation)

## Cross-Project Backlog Workflow

**Every project directory may contain a `BACKLOG.md`** with active work items in a standard format.

**When the user asks about priorities, daily planning, or "what should I work on":**

1. Glob search for `**/BACKLOG.md` across all working directories and under `~/Library/CloudStorage/OneDrive-Accenture/`
2. Read each BACKLOG.md found
3. Aggregate all items with status `Not Started` or `In Progress`
4. Sort by: overdue items first, then P1 > P2 > P3 > P4, then by due date
5. Present a cross-project daily view grouping by urgency (Overdue / Due Today / This Week / Upcoming)
6. Ask user which item to tackle first
7. After completing work, update the relevant BACKLOG.md (move Done items to Completed table)

**BACKLOG.md standard format:**

```
# [Project Name] — Backlog

## Active Items
| # | Work Item | Status | Priority | Due Date | Deliverable | Notes |

## Completed Items
| # | Work Item | Completed | Deliverable | Notes |
```

- **Status**: `Not Started` / `In Progress` / `Blocked` / `Done`
- **Priority**: `P1` (urgent) / `P2` (this week) / `P3` (this sprint) / `P4` (backlog)
- **Due Date**: ISO date (YYYY-MM-DD) or `—` if no hard deadline