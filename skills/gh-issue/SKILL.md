---
name: gh-issue
description: Create a single GitHub issue with the correct standard template for its type
argument-hint: <type> <title>
---

Creating a GitHub issue.

Type: $0
Title: $1

Use the template that matches the type. After creating, add to the project board.

---

## Templates

### epic
```bash
gh issue create --title "[EPIC] $1" --label "epic" \
  --body "## Goal\n[What problem does this epic solve?]\n\n## Success Criteria\n- [ ] ...\n\n## Child Features\n- [ ] (to be linked after creation)"
```

### feature
```bash
gh issue create --title "[FEATURE] $1" --label "feature" \
  --body "**Parent Epic:** #N\n\n## Acceptance Criteria\n- [ ] ...\n\n## Affected Files/Areas\n- backend/app/...\n- frontend/src/...\n\n## Dependencies\nBlocked by: #N or None\n\n## Notes\n..."
```

### story
```bash
gh issue create --title "[STORY] $1" --label "story" \
  --body "**Parent Feature:** #N\n\n## Task\n[Single, specific task — completable in 2–4 hrs]\n\n## Acceptance Criteria\n- [ ] ...\n\n## Affected Files/Areas\n- ..."
```

### bug
```bash
gh issue create --title "[BUG] $1" --label "bug" \
  --body "## Description\n[What is broken?]\n\n## Steps to Reproduce\n1. ...\n2. ...\n\n## Expected Behavior\n...\n\n## Actual Behavior\n...\n\n## Affected Files/Areas\n- ...\n\n## Severity\nP1 / P2 / P3"
```

### chore
```bash
gh issue create --title "[CHORE] $1" --label "chore" \
  --body "## Description\n[What non-functional work needs to be done?]\n\n## Acceptance Criteria\n- [ ] ...\n\n## Affected Files/Areas\n- ..."
```

---

## After Creating

Add to the GitHub project board:
```bash
gh project item-add [PROJECT_NUMBER] --owner [GITHUB_ORG] --url [ISSUE_URL]
```

Confirm the issue URL and project board link to the user.
