---
name: gh-review
description: Read all PR review comments, address feedback, and re-request review
argument-hint: <pr-number>
---

Reviewing and addressing feedback on PR #$ARGUMENTS.

---

## Step 1 — Read Everything First

```bash
gh pr view $ARGUMENTS --comments
gh pr diff $ARGUMENTS
```

Read all comments before making any changes. Do not start addressing feedback until you have a complete picture of everything being requested.

---

## Step 2 — Categorize Each Comment

For each comment, classify it as:
- **Must fix** — requested changes that block merge
- **Should address** — suggestions with clear merit worth implementing
- **Declining** — suggestions you will not implement (requires inline reply with reasoning)

Present this categorization to the user and wait for confirmation before making changes.

---

## Step 3 — Address Comments

Work through each comment in the "must fix" and "should address" categories.

For each declined suggestion, post an inline response:
```bash
gh pr comment $ARGUMENTS --body "Re: [comment topic] — Not implementing because [specific reasoning]. Happy to discuss if you feel strongly."
```

---

## Step 4 — Push Updates

Push all changes to the same branch. Do not open a new PR for review changes.

```bash
git push
```

---

## Step 5 — Re-request Review

```bash
gh pr review $ARGUMENTS --request-review $(gh pr view $ARGUMENTS --json reviewRequests -q '[.reviewRequests[].login] | join(",")')
```

If the reviewer list is empty, ask the user who to re-request from.

---

## Step 6 — Report to User

Summarize:
- What changes were made (by comment)
- What was declined and why
- PR status after push
