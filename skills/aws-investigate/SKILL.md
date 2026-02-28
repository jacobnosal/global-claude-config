---
name: aws-investigate
description: Investigate an AWS environment for errors, misconfigurations, and detective control gaps — using the current repo and Terraform state as context. Proposes fixes and files GitHub issues with findings.
argument-hint: (no arguments required)
---

Investigating AWS environment for errors and detective control gaps.

**This is a read-only investigation. Do not modify any files, AWS resources, or infrastructure.**

---

## Step 1 — Verify AWS Authentication

Run:
```bash
aws sts get-caller-identity
```

If this fails (expired SSO token, no active profile, missing credentials), **stop immediately** and ask the user to authenticate before continuing:

```bash
# SSO-based login (most common for enterprise accounts)
aws sso login --profile <profile-name>

# Or for static credentials
aws configure
```

Once authenticated, record and report:
- **Account ID**
- **Caller ARN** (role/user)
- **Active region** — check `AWS_DEFAULT_REGION`, then `aws configure get region`

Do not proceed to Step 2 until authentication is confirmed.

---

## Step 2 — Gather Repository Context

Identify the current repository and any Terraform artifacts that describe the AWS environment.

```bash
# Confirm repo root
git rev-parse --show-toplevel

# Recent commits for context
git log --oneline -10
```

Search for Terraform files:
- `**/*.tf` — resource definitions, provider configs (look for `provider "aws"` blocks to extract regions/accounts)
- `**/*.tfvars` — variable values (look for account IDs, region names, resource names)
- `terraform.tfstate` or `**/*.tfstate` — live state (if present locally)
- `.terraform/` directory — note if initialized

Read any `BACKLOG.md` or project README for operational context (known issues, environment names, recent deployments).

Note:
- Which AWS regions appear in the Terraform config
- Which resource types are managed (VPCs, ECS clusters, RDS, Lambda, etc.)
- Any references to detective controls (CloudTrail, GuardDuty, Config, Security Hub)
- Environment names (prod, staging, dev) and their account mappings

---

## Step 3 — Detective Controls Health Sweep

For each control below, check its status in **every region** referenced in the Terraform context (Step 2), plus `us-east-1` if not already included. For each check, note: enabled/disabled, region, and any issues observed.

### CloudTrail

```bash
# List all trails (non-shadow only)
aws cloudtrail describe-trails --include-shadow-trails false

# For each trail ARN returned, check its logging status
aws cloudtrail get-trail-status --name <trail-arn>

# Check if data events are captured (S3, Lambda)
aws cloudtrail get-event-selectors --trail-name <trail-arn>
```

Look for:
- `IsLogging: false` — trail exists but logging is stopped (HIGH)
- No trails found — CloudTrail entirely absent (CRITICAL)
- `IsMultiRegionTrail: false` — single-region only, blind spots in other regions (MEDIUM)
- No S3 or Lambda data event selectors — data-plane activity not captured (MEDIUM)

### GuardDuty

```bash
# List all detectors in the current region
aws guardduty list-detectors

# For each detector ID
aws guardduty get-detector --detector-id <detector-id>

# Fetch active HIGH/CRITICAL findings (severity >= 4.0)
aws guardduty list-findings \
  --detector-id <detector-id> \
  --finding-criteria '{
    "Criterion": {
      "severity": {"Gte": 4},
      "service.archived": {"Eq": ["false"]}
    }
  }'

# Fetch finding details for any IDs returned
aws guardduty get-findings \
  --detector-id <detector-id> \
  --finding-ids <id1> <id2>
```

Look for:
- No detectors found — GuardDuty not enabled (CRITICAL)
- `Status: DISABLED` — detector exists but paused (HIGH)
- Active findings with severity >= 7.0 (CRITICAL)
- Active findings with severity 4.0–6.9 (HIGH)

### AWS Config

```bash
# Check recorder status
aws configservice describe-configuration-recorders
aws configservice describe-configuration-recorder-status

# Check for non-compliant rules
aws configservice describe-compliance-by-config-rule \
  --compliance-types NON_COMPLIANT

# Check delivery channel
aws configservice describe-delivery-channels
aws configservice describe-delivery-channel-status
```

Look for:
- `recording: false` — recorder exists but not running (HIGH)
- No recorders found — Config not set up (HIGH)
- `lastStatus: FAILED` on delivery channel — findings not being delivered (MEDIUM)
- Non-compliant rules — note the rule names and affected resources

### Security Hub

```bash
# Check if Security Hub is enabled
aws securityhub describe-hub

# List enabled standards
aws securityhub get-enabled-standards

# Fetch active CRITICAL and HIGH findings
aws securityhub get-findings \
  --filters '{
    "RecordState": [{"Value": "ACTIVE", "Comparison": "EQUALS"}],
    "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}],
    "SeverityLabel": [
      {"Value": "CRITICAL", "Comparison": "EQUALS"},
      {"Value": "HIGH", "Comparison": "EQUALS"}
    ]
  }' \
  --max-items 25
```

Look for:
- `ResourceNotFoundException` or `InvalidAccessException` — Security Hub not subscribed (informational, note it)
- No standards enabled — Hub active but not evaluating anything (MEDIUM)
- CRITICAL/HIGH active findings — note the finding types and resources

### VPC Flow Logs

```bash
# List all VPCs
aws ec2 describe-vpcs --query 'Vpcs[*].{ID:VpcId,Name:Tags[?Key==`Name`].Value|[0],CIDR:CidrBlock}'

# For each VPC ID, check for flow logs
aws ec2 describe-flow-logs \
  --filter Name=resource-id,Values=<vpc-id>
```

Look for:
- VPCs with no flow logs — network traffic not captured (MEDIUM)
- Flow logs present but `FlowLogStatus: FAILED` — misconfigured delivery (MEDIUM)

---

## Step 4 — Active Alarms & Errors Sweep

### CloudWatch Alarms in ALARM State

```bash
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[*].{Name:AlarmName,Metric:MetricName,Namespace:Namespace,Reason:StateReason}'
```

For each alarm in ALARM state, note: alarm name, metric, namespace, and reason.

### AWS Health Events (Active)

```bash
# Note: Health API is global and must be called against us-east-1
aws health describe-events \
  --region us-east-1 \
  --filter '{"eventStatusCodes": ["open", "upcoming"]}' \
  --query 'events[*].{Service:service,Type:eventTypeCode,Region:region,Status:statusCode,StartTime:startTime}'
```

If this returns `SubscriptionRequiredException`, skip gracefully — Health API requires AWS Business or Enterprise Support. Note this in the summary but do not treat it as a finding.

### Recent Access Denied Errors (Last 24 Hours)

```bash
# macOS date format
START_TIME=$(date -u -v-24H +%Y-%m-%dT%H:%M:%SZ)

# Check for AccessDenied errors
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ErrorCode,AttributeValue=AccessDenied \
  --start-time "$START_TIME" \
  --max-results 20 \
  --query 'Events[*].{Time:EventTime,User:Username,Event:EventName,Source:EventSource}'

# Also check for UnauthorizedOperation errors
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ErrorCode,AttributeValue=UnauthorizedOperation \
  --start-time "$START_TIME" \
  --max-results 20 \
  --query 'Events[*].{Time:EventTime,User:Username,Event:EventName,Source:EventSource}'
```

Look for:
- Repeated `AccessDenied` from the same principal — possible misconfigured IAM or privilege escalation attempt
- `AccessDenied` on detective controls (e.g., `guardduty:GetDetector`) — something or someone trying to probe/disable controls
- Unexpected principals performing unexpected actions

---

## Step 5 — Analyze and Triage Findings

Synthesize all evidence from Steps 3–4. For each finding:

1. **Classify severity**
   | Severity | Examples |
   |----------|---------|
   | Critical | Detective control entirely absent in prod; active GuardDuty finding >= 7.0 |
   | High | Control exists but disabled/stopped; active finding 4.0–6.9; alarm actively firing |
   | Medium | Control partially configured; non-compliant Config rules; flow logs missing |
   | Low | Best-practice gap; informational health event; single-region trail in non-prod |

2. **Identify root cause** — misconfiguration, manual change bypassing IaC, drift, missing enablement, cost-saving shortcut, etc.

3. **Assess blast radius** — which accounts, regions, environments, or resource types are exposed?

4. **Link to Terraform** — if the finding relates to a resource in the Terraform state, identify which `.tf` file and resource block manages it. Note if there is a drift between the TF definition and the live state.

---

## Step 6 — Propose Fixes

For each finding, write a concrete, specific fix:
- Exact AWS CLI command to remediate, **or**
- Terraform resource block / attribute change needed
- Which file(s) to modify if IaC-managed
- Whether a one-time console/CLI action is needed vs. an IaC change
- Whether the fix must be applied in multiple regions

Do NOT implement the fix. Do NOT modify any files or AWS resources. This step is documentation only.

---

## Step 7 — File Investigation Summary Issue

Determine the current repo slug, then create one summary issue labeled `aws-investigation`:

```bash
REPO_SLUG=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')

gh issue create \
  --repo "$REPO_SLUG" \
  --title "[AWS Investigation] $(date +%Y-%m-%d) — <N> findings in <account-id>" \
  --label "aws-investigation" \
  --body "$(cat <<'EOF'
## Investigation Summary

- **Date:** <ISO date>
- **AWS Account:** <account-id>
- **Caller ARN:** <arn>
- **Region(s) Swept:** <comma-separated list>
- **Terraform Context:** <yes/no — files found>
- **Triggered By:** /aws-investigate

## Findings Overview

| # | Severity | Service | Finding | Issue |
|---|----------|---------|---------|-------|
| 1 | Critical | GuardDuty | Detector disabled in us-east-1 | #N |
| 2 | High | CloudTrail | Logging stopped on main trail | #N |
| 3 | Medium | VPC Flow Logs | vpc-abc123 missing flow logs | #N |

## Detective Controls Status

| Control | Status | Notes |
|---------|--------|-------|
| CloudTrail | ✅ / ⚠️ / ❌ | <summary> |
| GuardDuty | ✅ / ⚠️ / ❌ | <summary> |
| AWS Config | ✅ / ⚠️ / ❌ | <summary> |
| Security Hub | ✅ / ⚠️ / ❌ | <summary> |
| VPC Flow Logs | ✅ / ⚠️ / ❌ | <summary> |

## Active Alarms

<list alarms in ALARM state, or "None">

## AWS Health Events

<list open/upcoming events, or "None / Health API unavailable (requires Business Support)">

## Repository Context

- **Repo:** <repo slug>
- **Recent commits:** <last 3 commit messages>
- **Terraform state found:** <yes/no>
- **Managed resources:** <brief summary of resource types>
EOF
)"
```

Record the summary issue number — it will be referenced in each finding issue (`#N`).

---

## Step 8 — File One Issue Per Finding

For each finding from Step 5 (Critical through Medium; use judgment for Low), create a `bug` issue. File issues in order from highest to lowest severity.

```bash
gh issue create \
  --repo "$REPO_SLUG" \
  --title "[AWS] <Service>: <concise description of finding>" \
  --label "bug,aws-security" \
  --body "$(cat <<'EOF'
## Summary

<1–2 sentences: what is wrong, in which service, in which region/resource>

## Environment

- **AWS Account:** <account-id>
- **Region:** <region>
- **Service:** <service name>
- **Resource:** <resource ARN, ID, or name if applicable>

## Evidence

\`\`\`
<CLI output that confirms the problem — be specific, paste the relevant fields>
\`\`\`

## Root Cause

<Why this is happening — misconfiguration, IaC drift, manual change, missing enablement, etc.>

## Proposed Fix

<Concrete remediation steps. Choose whichever applies:>

**Option A — AWS CLI:**
\`\`\`bash
# Command to remediate
\`\`\`

**Option B — Terraform:**
\`\`\`hcl
# Resource block or attribute change needed
# File: <path/to/file.tf>
\`\`\`

**Option C — Console steps:**
1. Navigate to ...
2. ...

## Blast Radius

<What is exposed or at risk until this is resolved. Which environments, accounts, or workloads are affected.>

## Severity

Critical / High / Medium / Low — <1-sentence rationale>

## Related Investigation

Part of: #<summary-issue-number>
EOF
)"
```

After all issues are filed, output a final summary to the user:

```
Investigation complete.

Account:   <account-id>
Region(s): <regions>
Findings:  <N total> (<C> Critical, <H> High, <M> Medium, <L> Low)

Issues filed:
  #<N>  [Investigation summary] <url>
  #<N>  [Critical] <title> <url>
  #<N>  [High]     <title> <url>
  ...
```

---

## Constraints

- Do NOT modify any local repository files during this investigation
- Do NOT call any destructive or mutating AWS APIs (`delete`, `put`, `create`, `enable`, `disable`, etc.)
- Do NOT push commits, create branches, or modify existing GitHub issues or PRs
- Permitted write actions: creating GitHub issues in Steps 7 and 8 only
- If a service returns `SubscriptionRequiredException` or `AccessDeniedException`, skip it gracefully and note the gap — do not treat API unavailability as a finding unless the access denial itself is suspicious
- If Security Hub or GuardDuty is not subscribed/enabled, note it in the summary table but do not file a finding issue unless the repo's Terraform config clearly indicates it should be enabled
- Sweep all regions referenced in Terraform context; if no Terraform found, sweep the active region only and note the limited scope
