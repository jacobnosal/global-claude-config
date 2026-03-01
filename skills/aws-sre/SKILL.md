---
name: aws-sre
description: Investigate AWS SRE issues guided by local SRE.md runbooks — reads infrastructure tables, event taxonomies, sample queries, and alerting recommendations, then inspects the live environment for monitoring gaps, active incidents, and operational degradations.
argument-hint: [environment] (optional — e.g., prod, staging, dev; defaults to prod)
---

Investigating AWS environment for SRE issues using local runbook guidance.

**This is a read-only investigation. Do not modify any files, AWS resources, or infrastructure.**

---

## Step 0 — Parse Arguments

Extract the target environment from `$ARGUMENTS`. If no argument is provided, default to `prod`.

```
ENV="${ARGUMENTS:-prod}"
```

This value will be used throughout to resolve resource name patterns (e.g., `{env}` → `prod`).

---

## Step 1 — Find and Parse SRE.md Files

Search the current repository for all SRE runbooks:

```bash
# Glob for all SRE.md files (case-insensitive variants)
find . -type f \( -name "SRE.md" -o -name "sre.md" -o -name "SRE*.md" -o -name "runbook*.md" \) \
  ! -path "./.git/*" \
  | sort
```

Read every file found. For each SRE.md, extract and record the following structured information:

### 1a — Infrastructure Table

Find markdown tables under headings like "Infrastructure", "Resources", or "Architecture". For each row, resolve the `{env}` placeholder to `$ENV`:

- Resource type (e.g., RUM App Monitor, Cognito Identity Pool, IAM Role, Lambda, ECS Service, DynamoDB Table)
- Name pattern → resolved name (substituting `{env}` → the value from Step 0)
- ARN pattern (if provided)

Build a **Resource Registry** — a list of `(resource-type, resolved-name)` pairs that will drive discovery in Step 4.

### 1b — Alerting Recommendations

Find sections titled "Alerting" or "Alarms" or "Monitoring". For each recommended alarm, record:

- Metric name or event type being monitored
- Threshold (value, period, comparison)
- Expected alarm name (resolve `{env}` placeholder)
- Terraform file reference (if mentioned)

Build an **Expected Alarms List** — these will be verified in Step 6.

### 1c — Sample Queries

Find code blocks in sections titled "Querying", "Logs Insights", "CloudWatch Logs", or similar. Record:

- Log group name pattern (resolve `{env}`)
- Each query verbatim — these will be adapted and executed in Step 7

### 1d — Event Taxonomy

Find tables or lists describing custom events (e.g., `com.sce.*` namespaces, event types, fields). Record all event type strings — these will be used to check error rates in Step 7.

### 1e — Credential / Auth Chain

Find sections describing the auth or credential chain. Note any services involved (Cognito, STS, IAM roles) and which resources they are scoped to. These will be verified in Step 5.

### 1f — Known Limitations

Read any "Known Limitations" or "Caveats" sections. Record these to avoid false positives during analysis.

If **no SRE.md files are found**, report this to the user and stop:

```
No SRE.md runbooks found in this repository.
Expected locations: docs/SRE.md, SRE.md, runbooks/*.md
Create an SRE.md and re-run /aws-sre.
```

---

## Step 2 — Verify AWS Authentication

```bash
aws sts get-caller-identity
```

If authentication fails, stop immediately and ask the user to authenticate:

```bash
# SSO-based login (most common for enterprise accounts)
aws sso login --profile <profile-name>

# Or for static credentials
aws configure
```

Once authenticated, record and report:
- **Account ID**
- **Caller ARN**
- **Active region** — check `AWS_DEFAULT_REGION`, then `aws configure get region`

Do not proceed to Step 3 until authentication is confirmed.

---

## Step 3 — Gather Repository Context

```bash
git rev-parse --show-toplevel
git log --oneline -5
```

Search for Terraform files that define the resources named in the SRE.md:

```bash
# Find terraform files — look for resource definitions matching names from the Resource Registry
find . -name "*.tf" ! -path "./.git/*" | sort
```

For each Terraform file found:
- Note which resources from the Resource Registry are managed there
- Note which file manages the Expected Alarms (if any)
- Flag any `{env}` variable names used (these confirm environment naming conventions)

---

## Step 4 — Discover and Verify Infrastructure

For each entry in the Resource Registry (from Step 1a), verify the resource exists in AWS. Use the appropriate CLI command based on resource type:

### CloudWatch RUM App Monitor

```bash
# List all app monitors
aws rum list-app-monitors \
  --query 'AppMonitorSummaries[*].{Name:Name,State:State,Created:Created}'

# Get details for the specific monitor
aws rum get-app-monitor --name <resolved-monitor-name>
```

Note: `State: ACTIVE` is expected. `DELETING` or missing entirely is a finding.

### Cognito Identity Pool

```bash
# List identity pools (paginated — increase max-results if needed)
aws cognito-identity list-identity-pools --max-results 60 \
  --query 'IdentityPools[?contains(IdentityPoolName, `<resolved-pool-name-fragment>`)]'

# Get the specific pool
aws cognito-identity describe-identity-pool --identity-pool-id <id>
```

Check:
- `AllowUnauthenticatedIdentities: false` — if the SRE.md says unauthenticated must be disabled, flag `true` as HIGH
- Cognito provider ARN matches the expected user pool

### IAM Role

```bash
aws iam get-role --role-name <resolved-role-name>
aws iam list-attached-role-policies --role-name <resolved-role-name>
aws iam list-role-policies --role-name <resolved-role-name>
```

Check that the inline or attached policies match what the SRE.md describes (e.g., scoped to `rum:PutRumEvents` on a specific ARN only).

### Lambda Function

```bash
aws lambda get-function --function-name <resolved-name>
aws lambda get-function-configuration --function-name <resolved-name> \
  --query '{State:State,LastUpdateStatus:LastUpdateStatus,Runtime:Runtime,Timeout:Timeout,MemorySize:MemorySize}'
```

### ECS Service

```bash
aws ecs describe-services \
  --cluster <cluster-name> \
  --services <resolved-service-name> \
  --query 'services[*].{Status:status,Running:runningCount,Desired:desiredCount,Pending:pendingCount}'
```

### DynamoDB Table

```bash
aws dynamodb describe-table --table-name <resolved-name> \
  --query 'Table.{Status:TableStatus,Items:ItemCount,ReadCap:ProvisionedThroughput.ReadCapacityUnits}'
```

### API Gateway

```bash
aws apigateway get-rest-apis \
  --query 'items[?contains(name, `<name-fragment>`)]'
```

For each resource, record:
- **Found**: yes/no
- **Status**: active, degraded, missing
- **Matches SRE.md description**: yes/partial/no

Flag any resource in the Resource Registry that is **missing or in a non-healthy state** as a finding.

---

## Step 5 — Verify Credential and Auth Chain

For each service named in the credential chain (Step 1e):

### Cognito User Pool (if referenced)

```bash
aws cognito-idp list-user-pools --max-results 60 \
  --query 'UserPools[?contains(Name, `<name-fragment>`)]'
```

### STS — Validate role assumption is possible

```bash
# Check the trust policy on the IAM role used by the credential chain
aws iam get-role --role-name <resolved-role-name> \
  --query 'Role.AssumeRolePolicyDocument'
```

Verify the trust policy only permits the Cognito Identity service and only for authenticated identities.

### Scope Check — Verify role permissions are minimal

```bash
aws iam get-role-policy --role-name <resolved-role-name> --policy-name <policy-name>
```

Flag any permission that exceeds what the SRE.md documents (e.g., if the SRE.md says the role is scoped to `rum:PutRumEvents` only but the policy contains broader permissions).

---

## Step 6 — Check CloudWatch Alarms Against Expected Alarms List

For each alarm in the Expected Alarms List (from Step 1b):

```bash
# Check if the alarm exists and its current state
aws cloudwatch describe-alarms \
  --alarm-names "<resolved-alarm-name>" \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Metric:MetricName,Threshold:Threshold,Period:Period,Reason:StateReason}'
```

Record for each expected alarm:
- **Exists**: yes/no
- **State**: OK / ALARM / INSUFFICIENT_DATA
- **Configuration matches SRE.md threshold**: yes/no (compare period and threshold)

Then sweep for any alarms currently in ALARM state across the relevant namespace:

```bash
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[*].{Name:AlarmName,Metric:MetricName,Namespace:Namespace,Reason:StateReason,UpdatedAt:StateUpdatedTimestamp}'
```

Classify findings:
- Expected alarm **missing entirely** → MEDIUM (monitoring gap)
- Expected alarm exists but **misconfigured** (wrong threshold/period vs. SRE.md) → LOW
- Any alarm currently in **ALARM state** → HIGH (active incident)
- Expected alarm in **INSUFFICIENT_DATA** → LOW (alarm not receiving metrics — possible instrumentation gap)

---

## Step 7 — Run Operational Queries

### 7a — Run SRE.md Sample Queries

For each sample query collected in Step 1c, adapt it to the last 24 hours and execute it via CloudWatch Logs Insights:

```bash
LOG_GROUP="/aws/rum/<resolved-monitor-name>"   # or whatever the SRE.md specifies
START_TIME=$(date -u -v-24H +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

QUERY_ID=$(aws logs start-query \
  --log-group-name "$LOG_GROUP" \
  --start-time $(date -u -v-24H +%s) \
  --end-time $(date -u +%s) \
  --query-string '<paste adapted query here>' \
  --query 'queryId' --output text)

# Poll until complete
aws logs get-query-results --query-id "$QUERY_ID"
```

Run each sample query. Capture results and note:
- Any error event types with non-zero counts
- p95 latency values (flag if > 2000ms for CRUD operations, > 5000ms for file uploads)
- Upload failure counts (flag if > 0 in the last hour)
- API error rates (flag if any 5xx status codes appear)

If the log group does not exist, flag as MEDIUM (log delivery not configured on the app monitor).

### 7b — Error Rate Spot-Check

Using the event taxonomy from Step 1d, build a targeted error query:

```bash
# Count all error-type events in the last hour
QUERY_ID=$(aws logs start-query \
  --log-group-name "$LOG_GROUP" \
  --start-time $(date -u -v-1H +%s) \
  --end-time $(date -u +%s) \
  --query-string 'fields @timestamp, event.eventType
| filter event.eventType like /error/
| stats count() as error_count by event.eventType
| sort error_count desc' \
  --query 'queryId' --output text)

aws logs get-query-results --query-id "$QUERY_ID"
```

### 7c — Recent AWS Health Events

```bash
# Health API must be called against us-east-1
aws health describe-events \
  --region us-east-1 \
  --filter '{"eventStatusCodes": ["open", "upcoming"]}' \
  --query 'events[*].{Service:service,Type:eventTypeCode,Region:region,Status:statusCode,StartTime:startTime}'
```

Skip gracefully if `SubscriptionRequiredException` is returned — note it in the summary but do not treat as a finding.

---

## Step 8 — Check Recent Change Events (Last 24 Hours)

Correlate any degradation signals with recent infrastructure changes via CloudTrail:

```bash
START_TIME=$(date -u -v-24H +%Y-%m-%dT%H:%M:%SZ)

# Check for changes to monitored resources
aws cloudtrail lookup-events \
  --start-time "$START_TIME" \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<resolved-monitor-name> \
  --max-results 20 \
  --query 'Events[*].{Time:EventTime,User:Username,Event:EventName}'

# Check for IAM/Cognito role changes
aws cloudtrail lookup-events \
  --start-time "$START_TIME" \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<resolved-role-name> \
  --max-results 10 \
  --query 'Events[*].{Time:EventTime,User:Username,Event:EventName}'
```

Note any `Update*`, `Delete*`, `Put*`, or `Detach*` events — these are potential change correlation candidates.

---

## Step 9 — Analyze and Triage Findings

Synthesize all evidence from Steps 4–8. For each finding:

1. **Classify severity**

   | Severity | Examples |
   |----------|---------|
   | Critical | Core resource missing (app monitor, identity pool deleted); auth chain broken |
   | High     | Active CloudWatch alarm; sustained error rate above SRE.md threshold; credential scope exceeds documented minimum |
   | Medium   | Expected alarm missing (monitoring gap); log group missing (no log delivery); INSUFFICIENT_DATA on alarms |
   | Low      | Alarm misconfigured vs. SRE.md threshold; known limitation in SRE.md explains the observation |

2. **Cross-reference with Known Limitations** (Step 1f) — do not file issues for observations explicitly documented as expected gaps.

3. **Link to SRE.md section** — for each finding, cite the specific section of the SRE.md that defines the expected state or recommendation being violated.

4. **Correlate with recent changes** — if a degradation coincides with a CloudTrail event from Step 8, note the correlation.

---

## Step 10 — Propose Fixes

For each finding, write a concrete fix referencing the appropriate SRE.md section:

- Which AWS resource or Terraform file needs to change
- Exact CLI command to remediate, or Terraform resource/attribute to update
- Whether the fix is a one-time CLI action vs. an IaC change (note the Terraform file the SRE.md references, if any)
- Whether the SRE.md's alerting recommendations should be added to `terraform/monitoring.tf` (or equivalent)

Do NOT implement the fix. Do NOT modify any files or AWS resources. This step is documentation only.

---

## Step 11 — Present Findings for Review (Gate)

Before filing any GitHub issues, present the complete findings to the user in a structured summary and **wait for explicit approval**.

Output the following:

```
SRE Investigation — Findings Ready for Review
=============================================
Environment: <env>
Account:     <account-id>
Region:      <region>
SRE.md files: <list>

FINDINGS (<N total>)

  [Critical] <#>  <title>
             Resource: <name>
             Evidence: <1-line summary>
             Runbook ref: <SRE.md section>

  [High]     <#>  <title>
             ...

  [Medium]   <#>  <title>
             ...

  [Low]      <#>  <title>  (will be skipped unless you ask to include them)
             ...
```

Then ask:

> Which of these findings should I file as GitHub issues?
> Options:
>   A) All findings listed above
>   B) Only Critical and High
>   C) Let me choose — list the numbers to include (e.g., 1, 3, 5)
>   D) None — investigation complete, no issues needed

**Do not proceed to Step 12 until the user responds.** File only the issues the user approves. If the user selects option C, wait for the list of numbers before continuing.

---

## Step 12 — File Investigation Summary Issue (if user approved any findings)

Determine the current repo slug and create one summary issue:

```bash
REPO_SLUG=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')

gh issue create \
  --repo "$REPO_SLUG" \
  --title "[SRE Investigation] $(date +%Y-%m-%d) — $ENV — <N> findings" \
  --label "aws-sre" \
  --body "$(cat <<'EOF'
## SRE Investigation Summary

- **Date:** <ISO date>
- **Environment:** <env>
- **AWS Account:** <account-id>
- **Caller ARN:** <arn>
- **Region:** <region>
- **SRE.md files read:** <list of files>
- **Triggered By:** /aws-sre

## Findings Overview

| # | Severity | Category | Finding | Issue |
|---|----------|----------|---------|-------|
| 1 | High     | Active Incident | <description> | #N |
| 2 | Medium   | Monitoring Gap  | Expected alarm missing: <name> | #N |
| 3 | Low      | Config Drift    | Alarm threshold differs from SRE.md | #N |

## Infrastructure Health

| Resource | Expected Name | Found | Status | Notes |
|----------|--------------|-------|--------|-------|
| RUM App Monitor | <name> | ✅ / ❌ | Active / Missing | |
| Cognito Identity Pool | <name> | ✅ / ❌ | | |
| IAM Role | <name> | ✅ / ❌ | | |

## Monitoring Coverage

| Expected Alarm | Exists | State | Matches SRE.md | Notes |
|----------------|--------|-------|----------------|-------|
| <alarm-name> | ✅ / ❌ | OK / ALARM / INSUFFICIENT_DATA / Missing | ✅ / ⚠️ | |

## Operational Signals (Last 24h)

<Summary of query results: error counts, latency p95, upload failure rate, API error rate. Or "No log delivery configured.">

## Recent Changes

<List of CloudTrail events that may correlate with findings, or "None in last 24h.">

## SRE.md Coverage

- **Runbooks read:** <list>
- **Event types in taxonomy:** <count>
- **Sample queries executed:** <count>
- **Known limitations noted:** <count>
EOF
)"
```

Record the summary issue number — it will be referenced in each finding issue filed in Step 13.

---

## Step 13 — File One Issue Per Approved Finding

For each finding the user approved in Step 11, file a `bug` issue. Order from highest to lowest severity.

```bash
gh issue create \
  --repo "$REPO_SLUG" \
  --title "[SRE] <Category>: <concise description> ($ENV)" \
  --label "bug,aws-sre" \
  --body "$(cat <<'EOF'
## Summary

<1–2 sentences: what is wrong, in which resource, in which environment>

## Environment

- **AWS Account:** <account-id>
- **Region:** <region>
- **Environment:** <env>
- **Resource:** <name or ARN>
- **SRE.md Reference:** <section title and file path>

## Evidence

\`\`\`
<CLI output or query result that confirms the problem>
\`\`\`

## Root Cause

<Why this is happening — missing resource, monitoring gap, config drift, instrumentation gap, credential scope violation, etc.>

## Proposed Fix

<Concrete remediation steps per the SRE.md runbook. Choose whichever applies:>

**Option A — AWS CLI:**
\`\`\`bash
# Command to remediate
\`\`\`

**Option B — Terraform:**
\`\`\`hcl
# Resource block or attribute change
# File: <path/to/file.tf per SRE.md>
\`\`\`

**Option C — Console steps:**
1. Navigate to ...

## Runbook Reference

> <Quote the relevant SRE.md section that defines the expected state>
> Source: `<file path>` — `<section heading>`

## Blast Radius

<What is at risk until resolved. Which users, workflows, or data pipelines are affected.>

## Severity

Critical / High / Medium / Low — <1-sentence rationale citing the SRE.md threshold or expectation>

## Related Investigation

Part of: #<summary-issue-number>
EOF
)"
```

After all approved issues are filed, output a final summary:

```
SRE investigation complete.

Environment: <env>
Account:     <account-id>
Region:      <region>
SRE.md files: <list>
Findings:    <N total> (<C> Critical, <H> High, <M> Medium, <L> Low)
Filed:       <N approved> | Skipped: <N skipped>

Issues filed:
  #<N>  [Investigation summary]  <url>
  #<N>  [Critical] <title>       <url>
  #<N>  [High]     <title>       <url>
  ...

Skipped (not filed per your selection):
  - <title> (<severity>)
  ...
```

---

## Constraints

- Do NOT modify any local repository files during this investigation
- Do NOT call any destructive or mutating AWS APIs (`delete`, `put`, `create`, `enable`, `disable`, `update`, etc.)
- Do NOT push commits, create branches, or modify existing GitHub issues or PRs
- Permitted write actions: creating GitHub issues in Steps 12 and 13 only
- If a CloudWatch Logs Insights query returns no results, do not treat silence as confirmation of health — note that the log group may not be receiving events (this itself may be a finding)
- If a resource is listed as a Known Limitation in the SRE.md (e.g., "RUM is disabled in local development"), do not file a finding for expected gaps in non-production environments
- If no Terraform context is found, still complete the investigation — note in the summary that IaC-linked fix guidance is unavailable
- Sweep only the region active in the current AWS auth context unless the SRE.md references additional regions
- If `$ARGUMENTS` names a specific service or symptom (rather than an environment), interpret it as a scope filter — restrict the investigation to resources whose names contain that string, and note the narrowed scope in the summary issue
