---
name: gh-finish
description: Run all quality gates, verify Definition of Done, and open a PR for the current issue
---

Completing the current issue and opening a PR.

---

## Step 1 — Run Quality Gates

```bash
.claude/pre-push.sh
```

If `pre-push.sh` does not exist, run gates manually in this order (stop on first failure):

```bash
# Environment
podman-compose ps | grep -qE "Up|running" || { echo "ERROR: Stack not running — podman-compose up -d"; exit 1; }
[ -f .env ] || { echo "ERROR: .env missing — cp .env.example .env"; exit 1; }
podman-compose exec -T backend python -c "from app.core.config import settings; print('Config OK')"
podman-compose exec -T db    pg_isready -U postgres
podman-compose exec -T redis redis-cli ping
podman-compose exec -T backend alembic upgrade head

# Tests (70% coverage threshold is mandatory — do not remove --cov-fail-under)
podman-compose exec -T backend pytest tests/ -m 'not integration' --cov=app --cov-fail-under=70 --tb=short -q
podman-compose exec -T frontend npm run test -- --run

# Lint & types
podman-compose exec -T backend ruff check . && podman-compose exec -T backend ruff format --check .
podman-compose exec -T frontend npm run lint
podman-compose exec -T frontend npx tsc --noEmit

# Security
podman-compose exec -T backend bandit -r app/ -ll
podman-compose exec -T backend pip-audit -r requirements.txt
podman-compose exec -T frontend npm audit --audit-level=high --omit=dev
```

> **STOP if any gate fails.** Report the exact error output. Do not push. Do not suppress failures.

---

## Step 2 — Verify Definition of Done

Check every item. Do not open a PR until all are satisfied:

- [ ] All acceptance criteria in the issue are satisfied
- [ ] All quality gates passed (exit code 0, no suppressed failures)
- [ ] Test coverage ≥ 70%
- [ ] No new SKILL.md violations without a justification comment
- [ ] No new Medium/High security findings without a justification comment

> **STOP if any DoD item is not satisfied.** Report what is missing and why.

---

## Step 3 — Push and Open PR

```bash
BRANCH=$(git branch --show-current)
ISSUE_NUM=$(echo $BRANCH | grep -oE 'GH-[0-9]+' | grep -oE '[0-9]+')
ISSUE_TITLE=$(gh issue view ${ISSUE_NUM} --json title -q .title)

git push -u origin $BRANCH

gh pr create \
  --title "GH-${ISSUE_NUM}: ${ISSUE_TITLE}" \
  --body "Closes #${ISSUE_NUM}

## Changes
-

## Testing Notes


## Security Considerations
" \
  --base develop
```

---

## Step 4 — Update Project Board

```bash
PR_URL=$(gh pr view --json url -q .url)
gh issue comment ${ISSUE_NUM} --body "PR opened: ${PR_URL}"
```

Move the issue to "In Review" on the GitHub project board.

Report the PR URL to the user.
