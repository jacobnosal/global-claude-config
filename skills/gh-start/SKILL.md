---
name: gh-start
description: Start work on a GitHub issue — read full context, create branch, name the session, confirm scope before writing any code
argument-hint: <issue-number>
---

Starting work on issue #$ARGUMENTS.

## Steps

### 1. Read the issue in full
```bash
gh issue view $ARGUMENTS
```

### 2. Read project context
Open and read:
- `docs/SRS.md`
- `docs/architecture.md`
- Every file listed in the issue's "Affected Files/Areas" section
- Any SKILL.md files referenced by the affected subsystem

### 3. Check for conflicts
```bash
gh pr list --state open
git branch -a
```
Are there open PRs or branches that touch the same files? Flag any conflicts to the user.

### 4. Confirm scope
Summarize in 3–5 sentences exactly what you will build. List any ambiguities or missing information. Wait for user confirmation before writing any code.

### 5. Plan mode check

| Condition | Required action |
|-----------|----------------|
| Issue touches 3+ files or 2+ subsystems | Plan mode — propose full plan, wait for approval |
| DB schema change required | Plan mode |
| Scope is ambiguous after reading the issue | Plan mode |
| Single subsystem, clear scope | Proceed directly |

### 6. Create branch
```bash
git checkout develop && git pull
git checkout -b feature/GH-$ARGUMENTS-<slug>
```
Replace `<slug>` with a short kebab-case description derived from the issue title.

### 7. Name this session
Tell the user to run:
```
/rename GH-$ARGUMENTS-<slug>
```

---

Ready to implement. Confirm before writing the first line of code.
