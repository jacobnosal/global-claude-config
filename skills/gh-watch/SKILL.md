---
name: gh-watch
description: Monitor GitHub Actions workflow runs — watch checks on an open PR, post-merge workflows for a merged PR, or report on all running/recent workflows in the repo
argument-hint: [<pr-url> | <pr-number>]
---

Monitoring GitHub Actions: $ARGUMENTS

---

## Step 1 — Detect Repo Context

If `$ARGUMENTS` contains a GitHub URL, extract `OWNER`, `REPO`, and `PR_NUM` from it:
```
URL format: https://github.com/<OWNER>/<REPO>/pull/<PR_NUM>
```

If `$ARGUMENTS` is a bare number (e.g. `42`), detect the current repo:
```bash
REPO_SLUG=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
```
If that fails (not inside a git repo), ask the user which repo to watch before continuing.

If `$ARGUMENTS` is empty, detect the current repo the same way. If detection fails, ask the user.

---

## Step 2 — Route to the Correct Mode

**If a PR number or URL was provided:**
```bash
gh pr view "$PR_NUM" --repo "$REPO_SLUG" \
  --json number,title,state,mergedAt,headSha,headRefName,baseRefName,url
```

- PR `state` is `OPEN` → **Mode A: Monitor Open PR Checks**
- PR `state` is `MERGED` → **Mode B: Monitor Post-Merge Workflows**
- PR `state` is `CLOSED` (not merged) → Report that the PR was closed without merging; show any check results from when it was open, then stop.

**If no argument was provided:**
→ **Mode C: Repo-Wide Workflow Status**

---

## Mode A — Monitor Open PR Checks

### A1. Snapshot current check status
```bash
gh pr checks "$PR_NUM" --repo "$REPO_SLUG"
```

Present the results in a table:
```
PR #<number>: <title>
Branch: <headRefName> → <baseRefName>
Head: <headSha (short)>

| Check | Status | Conclusion | Details |
|-------|--------|------------|---------|
| ...   |        |            |         |

Overall: X passing · Y failing · Z pending
```

### A2. Watch until all checks complete
```bash
gh pr checks "$PR_NUM" --repo "$REPO_SLUG" --watch --interval 10
```

When all checks have settled (all pass or any fail), report the final summary:
- List any failing checks by name with their details URL
- State the overall result: **All checks passed** or **N check(s) failed**
- If checks failed, prompt: "Run `/gh-investigate <run-url>` on any failing run to diagnose the issue."

---

## Mode B — Monitor Post-Merge Workflows

### B1. Get the merge commit
```bash
gh pr view "$PR_NUM" --repo "$REPO_SLUG" \
  --json mergeCommit,mergedAt,baseRefName \
  --jq '{sha: .mergeCommit.oid, mergedAt: .mergedAt, base: .baseRefName}'
```

### B2. Find workflow runs triggered by the merge
```bash
# Try by commit SHA first
gh run list --repo "$REPO_SLUG" --commit "$MERGE_SHA" \
  --json databaseId,workflowName,status,conclusion,url,createdAt

# If empty, fall back to branch + time window (runs on base branch within 5 min of merge)
gh run list --repo "$REPO_SLUG" --branch "$BASE_BRANCH" --limit 20 \
  --json databaseId,workflowName,status,conclusion,url,createdAt,headSha
```

Filter to runs created at or after the merge timestamp.

### B3. Present the triggered run snapshot
```
PR #<number> merged to <base> at <mergedAt>
Merge commit: <sha (short)>

Triggered workflow runs:
| Workflow | Status | Conclusion | Run URL |
|----------|--------|------------|---------|
| ...      |        |            |         |
```

### B4. Watch all in-progress runs to completion
For each run with `status == "in_progress"` or `status == "queued"`:
```bash
gh run watch "$RUN_ID" --repo "$REPO_SLUG" --interval 10
```

Watch them sequentially (or report on each as it completes). After each run finishes, report:
- **Passed** or **Failed**
- Link to the run for details

When all runs have settled, output a final summary:
```
Post-merge workflow results:
  ✓ <workflow-name>  — passed (<duration>)
  ✗ <workflow-name>  — failed  → <run-url>

Overall: X passed · Y failed
```

If any run failed, prompt: "Run `/gh-investigate <run-url>` to diagnose the failure."

---

## Mode C — Repo-Wide Workflow Status

### C1. Fetch all active and recent runs
```bash
# In-progress and queued
gh run list --repo "$REPO_SLUG" --status "in_progress" --limit 20 \
  --json databaseId,workflowName,status,headBranch,event,createdAt,url

gh run list --repo "$REPO_SLUG" --status "queued" --limit 20 \
  --json databaseId,workflowName,status,headBranch,event,createdAt,url

# Recently completed (last 10)
gh run list --repo "$REPO_SLUG" --limit 10 \
  --json databaseId,workflowName,status,conclusion,headBranch,event,createdAt,url
```

### C2. Present the status report
```
Workflow Status — <REPO_SLUG>
Snapshot: <timestamp>

### In Progress / Queued
| Workflow | Branch | Trigger | Started | Run |
|----------|--------|---------|---------|-----|
| ...      |        |         |         |     |

### Recently Completed
| Workflow | Branch | Trigger | Result | Finished | Run |
|----------|--------|---------|--------|----------|-----|
| ...      |        |         |        |          |     |
```

If the "In Progress / Queued" section is empty, say so explicitly: "No workflows currently running or queued."

If any recently completed run has `conclusion == "failure"`, highlight it and prompt:
"Run `/gh-investigate <run-url>` to diagnose the failure."

### C3. Offer to watch
After presenting the snapshot, ask:
> "Would you like me to watch a specific run until it completes? Reply with a run ID or URL."

If the user provides one:
```bash
gh run watch "$RUN_ID" --repo "$REPO_SLUG" --interval 10
```

Report the final result when it settles.
