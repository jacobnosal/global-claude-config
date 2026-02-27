---
name: backlog-done
description: Mark a backlog item as complete — moves it from Active Items to Completed Items with today's date
argument-hint: <project-name> <item-number>
---

Marking a backlog item as done.

Arguments received: $ARGUMENTS

Parse the arguments: the first word is the project name, the second is the item number.

---

## Step 1 — Find the BACKLOG.md

Search for a BACKLOG.md matching the project name:
```bash
find ~/Library/CloudStorage/OneDrive-Accenture -name "BACKLOG.md" \
  -not -path "*/archive/*" 2>/dev/null
```

Match the file whose heading corresponds to the project name argument.

If no match is found, list available BACKLOG.md files and ask the user which to use.

---

## Step 2 — Find the Item

Read the BACKLOG.md. Locate the row in the Active Items table with the matching item number.

If the item number is not found, list all active items and ask the user to confirm which one.

---

## Step 3 — Confirm Before Writing

Show the user exactly what will change:

```
Marking as done:

Project: [project name]
Item #[N]: [Work Item description]

Will move from Active Items to Completed Items:
| [#] | [Work Item] | [today's date] | [Deliverable] | [Notes] |

Confirm?
```

Wait for explicit confirmation.

---

## Step 4 — Update the File

Make both changes in a single edit:

1. **Remove** the row from the Active Items table
2. **Append** a new row to the Completed Items table:

   | # | Work Item | Completed | Deliverable | Notes |

   - `#` — same number as the active item
   - `Work Item` — same description
   - `Completed` — today's date in ISO format (YYYY-MM-DD)
   - `Deliverable` — carry over from the active row
   - `Notes` — carry over from the active row, append any completion notes if provided

Do not reformat or reorder any other content in the file.

Confirm the write with the file path and item details.
