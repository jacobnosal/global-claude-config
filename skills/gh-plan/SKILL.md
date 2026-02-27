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

### Step 1 — Complexity Assessment

Score the work across these five dimensions:

| Dimension | Low (1) | Medium (2) | High (3) |
|-----------|---------|-----------|---------|
| Subsystems touched | 1 | 2–3 | 4+ |
| Estimated effort | < 4 hrs | 1–3 days | 1+ week |
| Independent deliverables | 1 | 2–4 | 5+ |
| External dependencies | None | 1 | 2+ |
| DB schema changes | No | Minor | Significant |

Score 5–7 → **Option A** · Score 8–11 → **Option B** · Score 12–15 → **Option C**

- **Option A:** 1 feature issue with acceptance criteria
- **Option B:** Parent epic + 2–6 independently shippable feature issues
- **Option C:** Full hierarchy — epic → features → stories (~2–4 hrs each)

---

### Step 2 — Present Recommendation

Output exactly this format before doing anything else:

```
Complexity score: [N/15]
Recommended: Option [A/B/C]
Reasoning: [2–3 sentences explaining why]

Proposed breakdown:
  Epic: [title]
    Feature 1: [title] — [one-line description]
    Feature 2: [title] — [one-line description]
    ...

Approve this breakdown?
```

Wait for explicit user approval. Do not proceed to Step 3 until approved.

---

### Step 3 — Create Issues (after approval only)

```bash
# Epic (Option B or C)
gh issue create --title "[EPIC] Title" --label "epic" \
  --body "## Goal\n...\n\n## Success Criteria\n- [ ] ...\n\n## Child Features\n- [ ] (to be linked)"

# Feature
gh issue create --title "[FEATURE] Title" --label "feature" \
  --body "**Parent Epic:** #N\n\n## Acceptance Criteria\n- [ ] ...\n\n## Affected Files/Areas\n- backend/...\n- frontend/...\n\n## Dependencies\nBlocked by: #N or None"

# Story (Option C only)
gh issue create --title "[STORY] Title" --label "story" \
  --body "**Parent Feature:** #N\n\n## Task\n...\n\n## Acceptance Criteria\n- [ ] ..."

# Add each issue to project board
gh project item-add [PROJECT_NUMBER] --owner [GITHUB_ORG] --url [ISSUE_URL]
```

Labels: `epic` · `feature` · `story` · `bug` · `chore` · `blocked`
