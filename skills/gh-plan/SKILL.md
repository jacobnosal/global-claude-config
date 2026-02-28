---
name: gh-plan
description: Assess complexity of a feature and recommend a GitHub issue hierarchy (epic/features/stories) before creating anything
argument-hint: <feature or work description>
---

You are planning new work for the current project.

Feature/work to plan: $ARGUMENTS

## Instructions

Do not create any GitHub issues until the user explicitly approves your recommendation.

---

### Step 1 — Project Context Questions

Before scoring complexity, ask the user these questions in a single message. Wait for answers before proceeding.

```
Before I build the issue hierarchy, I need a few quick answers:

1. **New or existing project?**
   (a) New project — no repo, no infra, nothing exists yet
   (b) Existing project — adding to something already running

2. **Infrastructure / cloud resources needed?**
   (a) Yes — needs IaC, cloud provisioning, containers, databases, secrets management, etc.
   (b) No — purely application-layer work

3. **CI/CD pipeline needed or affected?**
   (a) Yes — new pipeline or changes to existing (build, test, deploy, image publish, etc.)
   (b) No

4. **Developer bootstrap / onboarding needed?**
   (a) Yes — README, Makefile, dev-container, local setup scripts, environment docs
   (b) No

5. **GitHub Project board?**
   (a) Yes — create a new GitHub Project (v2) with table + roadmap views, populated fields
   (b) No — issues only

   If (a): Project owner? (org name or `@me`)
   If (a): Project name? (press Enter to default to the epic title)

Answer with the letter for each (e.g. "a, b, a, a, a") or free-text if it's more nuanced.
```

Capture the answers as: `new_project`, `needs_infra`, `needs_cicd`, `needs_bootstrap`, `create_project`, `project_owner`, `project_title`.

---

### Step 2 — Complexity Assessment

Score the work across these six dimensions (use context answers from Step 1):

| Dimension | Low (1) | Medium (2) | High (3) |
|-----------|---------|-----------|---------|
| Subsystems touched | 1 | 2–3 | 4+ |
| Estimated effort | < 4 hrs | 1–3 days | 1+ week |
| Independent deliverables | 1 | 2–4 | 5+ |
| External dependencies | None | 1 | 2+ |
| DB / data schema changes | No | Minor | Significant |
| Infrastructure changes | None | Moderate (1–2 resources) | Significant (IaC, pipelines, multi-env) |

Score 6–9 → **Option A** · Score 10–13 → **Option B** · Score 14–18 → **Option C**

- **Option A:** 1 feature issue with acceptance criteria
- **Option B:** Parent epic + 2–6 independently shippable feature issues
- **Option C:** Full hierarchy — epic → features → stories (~2–4 hrs each)

---

### Step 2a — Infra / CI/CD / Bootstrap Feature Injection

Based on Step 1 answers, append the following standard features to the breakdown **before presenting it**:

- `needs_infra = yes` → Add feature: **[FEATURE] Infrastructure Provisioning** — IaC, cloud resources, secrets, networking, environments
- `needs_cicd = yes` → Add feature: **[FEATURE] CI/CD Pipeline** — build, test, lint, image publish, deploy automation
- `needs_bootstrap = yes` OR `new_project = yes` → Add feature: **[FEATURE] Developer Bootstrap** — README, local dev setup, Makefile/scripts, environment docs, onboarding guide

Each injected feature gets its own issue and acceptance criteria. They are not optional or implied — they are explicit deliverables.

---

### Step 3 — Present Recommendation

Output exactly this format before doing anything else:

```
Complexity score: [N/18]
Recommended: Option [A/B/C]
Reasoning: [2–3 sentences explaining why]

Proposed breakdown:
  Epic: [title]
    Feature 1: [title] — [one-line description]
    Feature 2: [title] — [one-line description]
    ...
    [INFRA] Infrastructure Provisioning — (if applicable)
    [CI/CD] CI/CD Pipeline — (if applicable)
    [BOOTSTRAP] Developer Bootstrap — (if applicable)

Approve this breakdown?
```

Wait for explicit user approval. Do not proceed to Step 4 until approved.

---

### Step 4 — Create Issues with Native Sub-Issue Hierarchy (after approval only)

Issues must be created in order: Epic first, then Features, then Stories.
After creating each child issue, immediately link it to its parent using the GitHub sub-issues API.
Derive `OWNER` and `REPO` from the current repo: `gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'`

```bash
# Resolve repo coordinates
REPO_SLUG=$(gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"')

# Helper: get databaseId from issue number
# Usage: get_id <issue_number>
get_id() {
  gh issue view "$1" --json databaseId --jq '.databaseId'
}

# Helper: link child as sub-issue of parent
# Usage: link_sub <parent_number> <child_database_id>
link_sub() {
  gh api --method POST "/repos/$REPO_SLUG/issues/$1/sub_issues" \
    -F sub_issue_id="$2"
}

# ── Epic ──────────────────────────────────────────────────────────────────────
EPIC_URL=$(gh issue create --title "[EPIC] <title>" --label "epic" --body "$(cat <<'EOF'
## Goal
<goal>

## Success Criteria
- [ ] <criterion>

## Scope
This epic tracks all child features. See sub-issues for individual deliverables.
EOF
)")
EPIC_NUM=$(echo "$EPIC_URL" | grep -o '[0-9]*$')
EPIC_ID=$(get_id "$EPIC_NUM")

# ── Features (repeat block for each feature) ──────────────────────────────────
FEAT_URL=$(gh issue create --title "[FEATURE] <title>" --label "feature" --body "$(cat <<'EOF'
## Acceptance Criteria
- [ ] <criterion>

## Affected Areas
- <path or subsystem>

## Dependencies
Blocked by: #N or None
EOF
)")
FEAT_NUM=$(echo "$FEAT_URL" | grep -o '[0-9]*$')
FEAT_ID=$(get_id "$FEAT_NUM")
link_sub "$EPIC_NUM" "$FEAT_ID"   # ← attaches Feature under Epic

# ── Infrastructure Feature (if needs_infra = yes) ─────────────────────────────
INFRA_URL=$(gh issue create --title "[FEATURE] Infrastructure Provisioning" \
  --label "feature,infrastructure" --body "$(cat <<'EOF'
## Scope
- Cloud resources (compute, networking, storage)
- IaC (Terraform / Pulumi / Bicep)
- Secrets management
- Environment separation (dev / staging / prod)

## Acceptance Criteria
- [ ] All resources defined in IaC — no manual click-ops
- [ ] Secrets stored in vault, not in repo
- [ ] All environments are reproducible from code

## Dependencies
Blocked by: None
EOF
)")
INFRA_NUM=$(echo "$INFRA_URL" | grep -o '[0-9]*$')
INFRA_ID=$(get_id "$INFRA_NUM")
link_sub "$EPIC_NUM" "$INFRA_ID"

# ── CI/CD Feature (if needs_cicd = yes) ──────────────────────────────────────
CICD_URL=$(gh issue create --title "[FEATURE] CI/CD Pipeline" \
  --label "feature,cicd" --body "$(cat <<'EOF'
## Scope
- Lint and test gate on every PR
- Build and image publish on merge to main
- Deploy to staging on merge; deploy to prod on tag / manual approval

## Acceptance Criteria
- [ ] All PRs blocked by failing lint or tests
- [ ] Container image published to registry automatically
- [ ] Deploy to staging is fully automated (no manual steps)
- [ ] Pipeline config is version-controlled alongside app code

## Dependencies
Blocked by: None
EOF
)")
CICD_NUM=$(echo "$CICD_URL" | grep -o '[0-9]*$')
CICD_ID=$(get_id "$CICD_NUM")
link_sub "$EPIC_NUM" "$CICD_ID"

# ── Bootstrap Feature (if needs_bootstrap = yes OR new_project = yes) ─────────
BOOT_URL=$(gh issue create --title "[FEATURE] Developer Bootstrap" \
  --label "feature,dx" --body "$(cat <<'EOF'
## Scope
- README with project overview and quickstart
- Local dev setup (Makefile, scripts, or dev-container)
- Environment variable documentation (.env.example)
- Onboarding checklist for new contributors

## Acceptance Criteria
- [ ] New developer can run the app locally in < 10 min following README alone
- [ ] All required env vars documented with descriptions and example values
- [ ] Makefile / scripts cover: install, run, test, lint, build

## Dependencies
Blocked by: None
EOF
)")
BOOT_NUM=$(echo "$BOOT_URL" | grep -o '[0-9]*$')
BOOT_ID=$(get_id "$BOOT_NUM")
link_sub "$EPIC_NUM" "$BOOT_ID"

# ── Stories (Option C only — repeat per story, link to parent Feature) ─────────
STORY_URL=$(gh issue create --title "[STORY] <title>" --label "story" --body "$(cat <<'EOF'
## Task
<description>

## Acceptance Criteria
- [ ] <criterion>
EOF
)")
STORY_NUM=$(echo "$STORY_URL" | grep -o '[0-9]*$')
STORY_ID=$(get_id "$STORY_NUM")
link_sub "$FEAT_NUM" "$STORY_ID"   # ← attaches Story under its parent Feature

```

**Hierarchy result in GitHub UI:**
```
Epic
├── Feature A          ← native sub-issue of Epic
│   ├── Story A1       ← native sub-issue of Feature A (Option C only)
│   └── Story A2
├── Feature B
├── Infrastructure Provisioning
├── CI/CD Pipeline
└── Developer Bootstrap
```

Labels: `epic` · `feature` · `story` · `bug` · `chore` · `blocked` · `infrastructure` · `cicd` · `dx`

> **Note:** GitHub native sub-issues require GitHub Issues on a repo within a GitHub organization or a personal repo with the sub-issues feature enabled. If `link_sub` returns a 404, the feature may not be available for the repo — fall back to body-text cross-references (`**Parent Epic:** #N`).

---

### Step 5 — Create & Configure GitHub Project (only if `create_project = yes`)

Execute in order: create project → add fields → add issues → fetch item IDs → set field values.

```bash
OWNER="<project_owner>"           # org name or @me from Step 1 answer
PROJECT_TITLE="<project_title>"   # from Step 1 answer, or default to epic title

# ── 1. Create the project ─────────────────────────────────────────────────────
PROJECT_URL=$(gh project create --owner "$OWNER" --title "$PROJECT_TITLE")
PROJECT_NUM=$(echo "$PROJECT_URL" | grep -o '[0-9]*$')
PROJECT_ID=$(gh project view "$PROJECT_NUM" --owner "$OWNER" --format json --jq '.id')

echo "Project #$PROJECT_NUM created: $PROJECT_URL"

# ── 2. Add custom fields ──────────────────────────────────────────────────────
# (Status is built-in to GitHub Projects — do not recreate it)

gh project field-create "$PROJECT_NUM" --owner "$OWNER" \
  --name "Priority" --data-type "SINGLE_SELECT" \
  --single-select-options "P1 - Critical,P2 - High,P3 - Medium,P4 - Low"

gh project field-create "$PROJECT_NUM" --owner "$OWNER" \
  --name "Size" --data-type "SINGLE_SELECT" \
  --single-select-options "XS,S,M,L,XL"

gh project field-create "$PROJECT_NUM" --owner "$OWNER" \
  --name "Type" --data-type "SINGLE_SELECT" \
  --single-select-options "Epic,Feature,Story,Bug,Chore"

gh project field-create "$PROJECT_NUM" --owner "$OWNER" \
  --name "Start Date" --data-type "DATE"

gh project field-create "$PROJECT_NUM" --owner "$OWNER" \
  --name "Target Date" --data-type "DATE"

gh project field-create "$PROJECT_NUM" --owner "$OWNER" \
  --name "Sprint" --data-type "ITERATION"

# ── 3. Fetch all field IDs and option IDs in one call ─────────────────────────
FIELDS=$(gh project field-list "$PROJECT_NUM" --owner "$OWNER" --format json)

# Field node IDs
STATUS_FID=$(  echo "$FIELDS" | jq -r '.fields[] | select(.name == "Status")   | .id')
PRIORITY_FID=$(echo "$FIELDS" | jq -r '.fields[] | select(.name == "Priority") | .id')
SIZE_FID=$(    echo "$FIELDS" | jq -r '.fields[] | select(.name == "Size")     | .id')
TYPE_FID=$(    echo "$FIELDS" | jq -r '.fields[] | select(.name == "Type")     | .id')

# Status option IDs (built-in — names may differ; adjust if needed)
OPT_TODO=$(        echo "$FIELDS" | jq -r '.fields[] | select(.name=="Status") | .options[] | select(.name=="Todo")        | .id')
OPT_IN_PROGRESS=$( echo "$FIELDS" | jq -r '.fields[] | select(.name=="Status") | .options[] | select(.name=="In Progress") | .id')
OPT_DONE=$(        echo "$FIELDS" | jq -r '.fields[] | select(.name=="Status") | .options[] | select(.name=="Done")        | .id')

# Priority option IDs
OPT_P1=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Priority") | .options[] | select(.name=="P1 - Critical") | .id')
OPT_P2=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Priority") | .options[] | select(.name=="P2 - High")     | .id')
OPT_P3=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Priority") | .options[] | select(.name=="P3 - Medium")   | .id')
OPT_P4=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Priority") | .options[] | select(.name=="P4 - Low")      | .id')

# Size option IDs
OPT_XS=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Size") | .options[] | select(.name=="XS") | .id')
OPT_S=$( echo "$FIELDS" | jq -r '.fields[] | select(.name=="Size") | .options[] | select(.name=="S")  | .id')
OPT_M=$( echo "$FIELDS" | jq -r '.fields[] | select(.name=="Size") | .options[] | select(.name=="M")  | .id')
OPT_L=$( echo "$FIELDS" | jq -r '.fields[] | select(.name=="Size") | .options[] | select(.name=="L")  | .id')
OPT_XL=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Size") | .options[] | select(.name=="XL") | .id')

# Type option IDs
OPT_EPIC=$(   echo "$FIELDS" | jq -r '.fields[] | select(.name=="Type") | .options[] | select(.name=="Epic")    | .id')
OPT_FEATURE=$(echo "$FIELDS" | jq -r '.fields[] | select(.name=="Type") | .options[] | select(.name=="Feature") | .id')
OPT_STORY=$(  echo "$FIELDS" | jq -r '.fields[] | select(.name=="Type") | .options[] | select(.name=="Story")   | .id')

# ── 4. Add all issues to the project ─────────────────────────────────────────
# (repeat for every issue URL produced in Step 4)
gh project item-add "$PROJECT_NUM" --owner "$OWNER" --url "$EPIC_URL"
gh project item-add "$PROJECT_NUM" --owner "$OWNER" --url "$FEAT_URL"
# gh project item-add "$PROJECT_NUM" --owner "$OWNER" --url "$INFRA_URL"
# gh project item-add "$PROJECT_NUM" --owner "$OWNER" --url "$CICD_URL"
# gh project item-add "$PROJECT_NUM" --owner "$OWNER" --url "$BOOT_URL"
# gh project item-add "$PROJECT_NUM" --owner "$OWNER" --url "$STORY_URL"

# ── 5. Fetch all project items (to resolve item node IDs by issue number) ─────
ITEMS=$(gh project item-list "$PROJECT_NUM" --owner "$OWNER" --format json --limit 200)

# Helper: look up item node ID by issue number
item_id() { echo "$ITEMS" | jq -r --argjson n "$1" '.items[] | select(.content.number == $n) | .id'; }

# Helper: set a single-select field on a project item
set_select() {
  # $1=item_node_id  $2=field_node_id  $3=option_node_id
  gh project item-edit --project-id "$PROJECT_ID" --id "$1" \
    --field-id "$2" --single-select-option-id "$3"
}

# ── 6. Populate field values ──────────────────────────────────────────────────
# Default priority/size rationale:
#   Epic              → P1 / XL  (drives everything)
#   Infra / CI/CD     → P1 / L   (blocking; substantial effort)
#   Bootstrap         → P2 / M
#   Feature           → P2 / M
#   Story             → P3 / S

# Epic
ITEM=$(item_id "$EPIC_NUM")
set_select "$ITEM" "$STATUS_FID"   "$OPT_TODO"
set_select "$ITEM" "$PRIORITY_FID" "$OPT_P1"
set_select "$ITEM" "$SIZE_FID"     "$OPT_XL"
set_select "$ITEM" "$TYPE_FID"     "$OPT_EPIC"

# Feature (repeat block for each application feature)
ITEM=$(item_id "$FEAT_NUM")
set_select "$ITEM" "$STATUS_FID"   "$OPT_TODO"
set_select "$ITEM" "$PRIORITY_FID" "$OPT_P2"
set_select "$ITEM" "$SIZE_FID"     "$OPT_M"
set_select "$ITEM" "$TYPE_FID"     "$OPT_FEATURE"

# Infrastructure feature
ITEM=$(item_id "$INFRA_NUM")
set_select "$ITEM" "$STATUS_FID"   "$OPT_TODO"
set_select "$ITEM" "$PRIORITY_FID" "$OPT_P1"
set_select "$ITEM" "$SIZE_FID"     "$OPT_L"
set_select "$ITEM" "$TYPE_FID"     "$OPT_FEATURE"

# CI/CD feature
ITEM=$(item_id "$CICD_NUM")
set_select "$ITEM" "$STATUS_FID"   "$OPT_TODO"
set_select "$ITEM" "$PRIORITY_FID" "$OPT_P1"
set_select "$ITEM" "$SIZE_FID"     "$OPT_L"
set_select "$ITEM" "$TYPE_FID"     "$OPT_FEATURE"

# Bootstrap feature
ITEM=$(item_id "$BOOT_NUM")
set_select "$ITEM" "$STATUS_FID"   "$OPT_TODO"
set_select "$ITEM" "$PRIORITY_FID" "$OPT_P2"
set_select "$ITEM" "$SIZE_FID"     "$OPT_M"
set_select "$ITEM" "$TYPE_FID"     "$OPT_FEATURE"

# Story (repeat block for each story)
ITEM=$(item_id "$STORY_NUM")
set_select "$ITEM" "$STATUS_FID"   "$OPT_TODO"
set_select "$ITEM" "$PRIORITY_FID" "$OPT_P3"
set_select "$ITEM" "$SIZE_FID"     "$OPT_S"
set_select "$ITEM" "$TYPE_FID"     "$OPT_STORY"
```

#### After the script completes — manual roadmap configuration

The roadmap view layout cannot be fully configured via `gh` CLI. Instruct the user:

```
Project created: <PROJECT_URL>

To configure the Roadmap view:
1. Open the project → click "+ New view" → select "Roadmap"
2. In the roadmap view settings (⚙):
   - Set "Date field" → Start Date
   - Set "End date field" → Target Date
   - (Optional) Group by → Sprint for iteration-based view
3. Set Start Date and Target Date on each item in the roadmap view
   — these cannot be populated automatically without known delivery dates.
4. Review and adjust Priority and Size fields — defaults were set
   conservatively; tune them to match your actual plan.

Table view fields populated automatically:
  ✓ Status     → Todo (all)
  ✓ Type       → Epic / Feature / Story
  ✓ Priority   → P1 / P2 / P3 (by type — review and adjust)
  ✓ Size       → XS–XL (by type — review and adjust)
  ✗ Start Date → set manually in roadmap view
  ✗ Target Date → set manually in roadmap view
  ✗ Sprint     → assign via project Sprint field once iterations are defined
```
