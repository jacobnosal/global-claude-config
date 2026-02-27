---
name: backlog-add
description: Add a new work item to a project's BACKLOG.md active items table
argument-hint: <project-name> <item description>
---

Adding a new item to the backlog.

Arguments received: $ARGUMENTS

Parse the arguments: the first word is the project name, everything after is the item description.

---

## Step 1 — Find the BACKLOG.md

Search for a BACKLOG.md that matches the project name:
```bash
find ~/Library/CloudStorage/OneDrive-Accenture -name "BACKLOG.md" \
  -not -path "*/archive/*" 2>/dev/null
```

Match the file whose heading (`# [Project Name] — Backlog`) corresponds to the project name argument.

If no match is found, list the BACKLOG.md files that were found and ask the user which one to use.
If no BACKLOG.md files exist at all, ask the user for the full path.

---

## Step 2 — Gather Missing Fields

Read the BACKLOG.md to determine the next item number (highest existing # + 1).

If any of the following were not provided in the arguments, ask for them before proceeding:
- **Priority:** P1 (urgent) / P2 (this week) / P3 (this sprint) / P4 (backlog)
- **Due Date:** ISO date (YYYY-MM-DD) or `—` if no hard deadline
- **Deliverable:** What the output of this work item is (document, PR, presentation, etc.)
- **Notes:** Optional — any context worth capturing

Default Status: `Not Started`

---

## Step 3 — Confirm Before Writing

Present the new row to the user before making any changes:

```
New item to add:

Project: [project name]
File: [path to BACKLOG.md]

| [#] | [Work Item] | Not Started | [Priority] | [Due Date] | [Deliverable] | [Notes] |

Add this item?
```

Wait for explicit confirmation.

---

## Step 4 — Add the Row

Append the new row to the Active Items table in the BACKLOG.md. Preserve all existing rows and formatting exactly. Do not reformat or reorder existing content.

Confirm the write to the user with the file path and item number.
