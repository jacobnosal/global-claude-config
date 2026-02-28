---
name: gh-investigate
description: Investigate a broken GitHub Actions workflow run, determine root cause, propose a fix, and file a GitHub issue with findings
argument-hint: <github-actions-run-or-job-url>
---

Investigating broken GitHub Actions workflow run: $ARGUMENTS

**This is a read-only investigation. Do not modify any files in the repository.**

---

## Step 1 — Parse the URL

From the URL, extract:

```
URL format: https://github.com/<OWNER>/<REPO>/actions/runs/<RUN_ID>
         or https://github.com/<OWNER>/<REPO>/actions/runs/<RUN_ID>/job/<JOB_ID>
```

Set shell variables:
```bash
URL="$ARGUMENTS"
OWNER=$(echo "$URL" | sed 's|https://github.com/||' | cut -d'/' -f1)
REPO=$(echo "$URL"  | sed 's|https://github.com/||' | cut -d'/' -f2)
RUN_ID=$(echo "$URL" | grep -oE 'runs/[0-9]+' | grep -oE '[0-9]+')
JOB_ID=$(echo "$URL" | grep -oE 'job/[0-9]+'  | grep -oE '[0-9]+')
REPO_SLUG="$OWNER/$REPO"
```

---

## Step 2 — Fetch Workflow Run Overview

```bash
gh run view "$RUN_ID" --repo "$REPO_SLUG"
```

Note: the triggering event, branch/commit, workflow name, overall status, and which jobs failed.

---

## Step 3 — Fetch Failed Job Logs

If a specific JOB_ID was provided in the URL:
```bash
gh run view --job "$JOB_ID" --repo "$REPO_SLUG" --log
```

Otherwise, fetch logs for all failed steps:
```bash
gh run view "$RUN_ID" --repo "$REPO_SLUG" --log-failed
```

Read the full log output. Note:
- The exact step that failed
- The error message(s)
- Any preceding warnings or setup failures
- Exit codes

---

## Step 4 — Read the Workflow Definition

Find and read the workflow YAML that triggered this run:
```bash
gh run view "$RUN_ID" --repo "$REPO_SLUG" --json workflowName --jq '.workflowName'
```

Then fetch the workflow file contents:
```bash
gh api "repos/$REPO_SLUG/contents/.github/workflows" --jq '.[].name'
```

Match the workflow name to a file, then read it:
```bash
gh api "repos/$REPO_SLUG/contents/.github/workflows/<workflow-file>.yml" --jq '.content' | base64 --decode
```

Read the full workflow YAML. Note:
- The job definition that failed
- Step ordering and dependencies
- Environment variables, secrets references, `runs-on`, matrix strategy
- Any `if:` conditions that may affect execution
- Pinned action versions

---

## Step 5 — Read Supporting Context (as needed)

If the failure references a specific file (Dockerfile, script, config), read it via the API — do not check out or modify any files locally:
```bash
gh api "repos/$REPO_SLUG/contents/<path>" --jq '.content' | base64 --decode
```

Check recent commits on the branch that triggered the run to see what changed:
```bash
gh api "repos/$REPO_SLUG/commits?sha=<branch>&per_page=5" --jq '.[].commit.message'
```

---

## Step 6 — Analyze and Determine Root Cause

Synthesize all evidence from Steps 2–5. Identify:

1. **Proximate cause** — the exact step, command, or condition that produced the failure
2. **Root cause** — the underlying reason (misconfiguration, dependency version pin, missing secret, timing issue, API change, etc.)
3. **Scope** — is this a one-off flake or a systemic problem?
4. **Blast radius** — does this affect other workflows, environments, or branches?

---

## Step 7 — Propose a Fix

Write a concrete, specific fix:
- Which file(s) need to change and how
- Whether a secret, variable, or permission needs to be added/updated
- Whether a dependency version needs pinning or unpinning
- Whether the fix needs to be applied to other workflows too

Do NOT implement the fix. Do NOT modify any files. The fix is documentation only.

---

## Step 8 — File a GitHub Issue

Create a bug issue in the repository with full investigation findings:

```bash
gh issue create \
  --repo "$REPO_SLUG" \
  --title "[BUG] CI failure: <short description of failure>" \
  --label "bug,ci" \
  --body "$(cat <<'EOF'
## Summary

<1–2 sentence description of what is broken and where>

## Failing Workflow Run

- **Run:** $URL
- **Workflow:** <workflow name>
- **Job:** <job name>
- **Step:** <step name>
- **Branch/Commit:** <branch> @ <sha>

## Error Output

\`\`\`
<paste the key error lines from the log>
\`\`\`

## Root Cause

<Clear explanation of why this is failing. Reference specific lines in the workflow YAML or supporting files.>

## Proposed Fix

<Concrete description of the change needed — file path, what to change, and why.>

## Affected Files

- `.github/workflows/<workflow>.yml`
- <any other files>

## Severity

P1 / P2 / P3 — <brief rationale>
EOF
)"
```

Report the issue URL to the user.

---

## Constraints

- Do NOT modify any repository files during this investigation
- Do NOT push any commits or create any branches
- Do NOT close or comment on existing issues or PRs
- The only write action permitted is creating the one GitHub issue in Step 8
