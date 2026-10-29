---
name: pr-plan-review-comments
description: Use when a pull request has review comments to fetch, triage, and either turn into a fix plan or fix and reply to. Covers inline threads and bot review bodies.
---

# PR Review Comments

Fetch unresolved review threads and actionable review bodies, triage which ones still hold, then either plan the fixes or apply them.

## Modes

- **Plan** — produce an implementation plan for the valid comments. Use `writing-plans`.
- **Apply** — fix the valid comments now, then commit, push, and reply through `commit-push-reply-comments`.

Ask which mode when the request is ambiguous.

## 1. Identify the PR

```bash
gh pr view --json number,url,headRefName
```

If the user names a PR, use it. In a monorepo with git submodules, run `gh` from the submodule that owns the PR.

## 2. Fetch review threads, reviews, and conversation comments

Run the bundled script:

```bash
python scripts/fetch_comments.py > pr_comments.json
```

The script shells out to `gh api graphql` and returns conversation comments, reviews, and review threads with resolution status.

Fetch the latest `metal-agent-ops` review separately:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate \
  --jq '[.[] | select(.user.login == "metal-agent-ops")] \
    | sort_by(.submitted_at) | last \
    | {id, state, submitted_at, commit_id, body}'
```

Keep only threads where `isResolved == false`, and treat every actionable review-body finding as a triage item.

Before planning or applying, re-fetch the head and update time:

```bash
gh pr view <pr> --json headRefOid,updatedAt
```

If either changed, repeat the fetches. This prevents acting on stale review data.

## 3. Triage

`isResolved: false` alone does not prove a comment is unaddressed. For each unresolved thread, read the current code at `path`/`line` and classify:

- **Already fixed** — a later commit changed the flagged code. Recommend resolving the thread.
- **Still valid** — the issue reproduces today.
- **Not worth acting on** — style nit, bot noise, or a deliberate decision.

Deduplicate `metal-agent-ops` review-body findings against inline findings that name the same defect. Compare the reviewed commit with the PR head before classifying.

If the branch is not checked out locally:

```bash
git fetch origin pull/<n>/head:pr-<n>-inspect
git show pr-<n>-inspect:<path>
git log pr-<n>-inspect --oneline -- <path>
git branch -D pr-<n>-inspect
```

If validity is ambiguous, ask the user rather than guessing.

Present the triage as a numbered list before planning or fixing:

```
1. [file:line] "Comment summary..." — Still valid: <reason>
2. [file:line] "Comment summary..." — Already fixed in <commit>, recommend resolving
3. [file:line] "Comment summary..." — Not worth acting on: <reason>
```

## 4a. Plan mode

**REQUIRED SUB-SKILL:** Use `writing-plans`. Save the plan under `docs/superpowers/plans/` following its naming convention. Each task traces to one triaged "still valid" comment or a small tightly-related cluster, and links the review thread's `url`. Present the plan inline as well.

## 4b. Apply mode

Fix the still-valid comments. Use `test-driven-development` and verify before pushing.

Then, **REQUIRED SUB-SKILL:** Use `commit-push-reply-comments` to commit, push, and reply. That skill owns the reply format and the full-SHA verification.

## Common mistakes

| Mistake | Fix |
| --- | --- |
| Trusting `isResolved` alone | Cross-check the current code and commit history. |
| Reading only inline threads | Process `metal-agent-ops` review bodies too. |
| Acting on stale review data | Re-fetch when the head or update time changed. |
| Using the REST `pulls/{n}/comments` endpoint | Resolution is a GraphQL `reviewThread` field. |
| Acting on stale, already-fixed, or nit comments | Triage first. |
| Using a short or local-only SHA in a reply | Verify the full SHA from the pushed PR. |
