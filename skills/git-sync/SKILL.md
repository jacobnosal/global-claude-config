---
name: git-sync
description: Sync the current feature branch with develop via rebase, resolve any conflicts, then run quality gates
---

Syncing current branch with develop.

---

## Step 1 — Fetch and Rebase

```bash
git fetch origin
git rebase origin/develop
```

---

## Step 2 — Resolve Conflicts (if any)

If conflicts arise, list every conflicted file:
```bash
git diff --name-only --diff-filter=U
```

Resolve each file manually. After resolving all files:
```bash
git add <resolved-file>
git rebase --continue
```

Do not use `git rebase --skip` unless the commit being skipped is genuinely empty after resolution. Report any skipped commits to the user.

---

## Step 3 — Run Quality Gates

After a clean rebase (with or without conflicts), run quality gates to verify nothing broke:

```bash
.claude/pre-push.sh
```

If `pre-push.sh` is not present, run at minimum:
```bash
podman-compose exec -T backend pytest tests/ -m 'not integration' --cov=app --cov-fail-under=70 --tb=short -q
podman-compose exec -T frontend npm run test -- --run
podman-compose exec -T backend ruff check .
podman-compose exec -T frontend npm run lint
```

> **STOP if any gate fails after rebase.** The rebase may have introduced a regression. Report the failure before proceeding.

---

## Step 4 — Report

Summarize:
- How many commits were rebased
- Whether there were conflicts and which files were affected
- Quality gate result (pass / fail with details)
