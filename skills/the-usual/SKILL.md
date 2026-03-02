---
name: the-usual
description: Full git→GitHub lifecycle — start an issue (with number) or commit+push+PR (no args). State-aware orchestrator for the standard workflow.
argument-hint: [issue-number]
---

Running the usual workflow.

---

## Determine Phase

```bash
git branch --show-current
git status --short
gh pr list --head $(git branch --show-current) --json number,url --jq '.[0]'
```

**If `$ARGUMENTS` is provided (an issue number):** go to **Phase 1 — Start**.
**If no arguments and on a feature branch with changes:** go to **Phase 2 — Commit + Finish**.
**If no arguments and on main/develop:** stop and tell the user to provide an issue number.

---

## Phase 1 — Start (called as `/the-usual <issue-number>`)

### 1.1 Read the issue
```bash
gh issue view $ARGUMENTS
```

### 1.2 Read project context
Open and read:
- `docs/SRS.md` (if present)
- `docs/architecture.md` (if present)
- Every file listed in the issue's "Affected Files/Areas" section
- Any `SKILL.md` referenced by the affected subsystem

### 1.3 Check for conflicts
```bash
gh pr list --state open
git branch -a
```
Flag any open PRs or branches touching the same files.

### 1.4 Confirm scope
Summarize in 3–5 sentences exactly what will be built. List ambiguities. **Wait for user confirmation before creating the branch.**

### 1.5 Create branch and name the session
```bash
git checkout main && git pull
git checkout -b feature/GH-$ARGUMENTS-<slug>
```
Replace `<slug>` with a short kebab-case description from the issue title.

Tell the user to run:
```
/rename GH-$ARGUMENTS-<slug>
```

**Stop here.** Do the work, then run `/the-usual` (no arguments) when ready to ship.

---

## Phase 2 — Commit + Push + PR (called as `/the-usual` on a feature branch)

### 2.1 Run quality gates

```bash
.claude/pre-push.sh
```

If `pre-push.sh` does not exist, run gates manually (stop on first failure):

```bash
# Environment
podman-compose ps | grep -qE "Up|running" || { echo "ERROR: Stack not running"; exit 1; }

# Tests
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

> **STOP if any gate fails.** Report the exact error. Do not proceed. Do not suppress.

### 2.2 Verify Definition of Done

- [ ] All acceptance criteria in the issue are satisfied
- [ ] All quality gates passed (exit code 0)
- [ ] Test coverage ≥ 70%
- [ ] No new Medium/High security findings without justification

> **STOP if any DoD item is not satisfied.** Report what is missing.

### 2.3 Stage changes

```bash
git status
git diff --stat
git add -A
```

### 2.4 Commit and push

Read the staged diff and draft a [Conventional Commits](https://www.conventionalcommits.org/) message:
```bash
git diff --cached
```

- `feat(scope): ...` for new functionality
- `fix(scope): ...` for bug fixes
- `refactor(scope): ...` for non-behaviour-changing changes
- `chore(scope): ...` for tooling/config/maintenance
- `test(scope): ...` for test-only changes
- `docs(scope): ...` for documentation-only changes

Present the message to the user and wait for approval (or edits) before committing.

```bash
git commit -m "<approved message>"
git push -u origin $(git branch --show-current)
```

### 2.5 Open the PR

```bash
BRANCH=$(git branch --show-current)
ISSUE_NUM=$(echo $BRANCH | grep -oE 'GH-[0-9]+' | grep -oE '[0-9]+')
ISSUE_TITLE=$(gh issue view ${ISSUE_NUM} --json title -q .title)

gh pr create \
  --title "GH-${ISSUE_NUM}: ${ISSUE_TITLE}" \
  --body "Closes #${ISSUE_NUM}

## Changes
-

## Testing Notes


## Security Considerations
" \
  --base main
```

### 2.6 Link PR to issue

```bash
PR_URL=$(gh pr view --json url -q .url)
gh issue comment ${ISSUE_NUM} --body "PR opened: ${PR_URL}"
```

Report the PR URL to the user.

### 2.7 Watch checks

Run `/gh-watch` to monitor the CI checks on the newly opened PR and report results when complete.

---

## Quick Reference

| When | Command |
|------|---------|
| Starting a new issue | `/the-usual <issue-number>` |
| Done coding, ready to ship | `/the-usual` |
| Need to sync with main first | `/git-sync` then `/the-usual` |
| Addressing PR review feedback | `/gh-review <pr-number>` |
