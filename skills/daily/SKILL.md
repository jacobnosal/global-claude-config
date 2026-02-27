---
name: daily
description: Aggregate all BACKLOG.md files across projects, sort by priority and urgency, and present today's prioritized work plan
---

Running daily planning view.

---

## Step 1 — Find All BACKLOG.md Files

Search this location, excluding the archive directory:
```bash
find ~/Library/CloudStorage/OneDrive-Accenture -name "BACKLOG.md" \
  -not -path "*/archive/*" 2>/dev/null
```

Read every BACKLOG.md found. Note the project name from each file's heading.

---

## Step 2 — Check for Priorities File

If this file exists, read it for career-level priority alignment:
```
~/Library/CloudStorage/OneDrive-Accenture/projects/trackers/PRIORITIES.md
```

---

## Step 3 — Aggregate Active Items

Collect every row where Status is `Not Started` or `In Progress` across all BACKLOG.md files.
Record the source project name alongside each item.

---

## Step 4 — Sort and Group

Today's date: use the current date from the system.

**Sort order within each group:**
1. P1 before P2 before P3 before P4
2. Earlier due date before later due date within the same priority
3. `In Progress` before `Not Started` within the same priority + date

**Groups:**
- **Overdue** — due date is before today
- **Due Today** — due date is today
- **This Week** — due date is within 7 days from today
- **Upcoming** — due date is more than 7 days out or `—`

---

## Step 5 — Present the View

Output in this format:

```
## Daily Planning — [YYYY-MM-DD]

### 🔴 Overdue
| Project | # | Work Item | Priority | Due | Status |
|---------|---|-----------|----------|-----|--------|
| ...     |   |           |          |     |        |

### 🟡 Due Today
| Project | # | Work Item | Priority | Due | Status |
| ...

### 🔵 This Week
| Project | # | Work Item | Priority | Due | Status |
| ...

### ⚪ Upcoming
| Project | # | Work Item | Priority | Due | Status |
| ...
```

If a group is empty, omit it entirely.

If PRIORITIES.md was found and any items align with career priorities, note them with a `★` in the Work Item column.

---

## Step 6 — Ask What to Tackle

After presenting the view:

> "Which item do you want to tackle first? Reply with `<project-name> #<number>` or describe what you want to work on."

When the user selects an item:
- If its status is `Not Started`, update it to `In Progress` in the relevant BACKLOG.md
- Confirm the update with the user before writing
