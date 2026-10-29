---
name: pr-agent-review-loop
description: Use when an open PR should get the agent reviewer (metal-agent-ops) to review, or when review feedback must be triaged, fixed, pushed, and replied to until the PR is approved. Covers coordinated frontend/backend repos linked to a Linear ticket.
---

# PR Agent Review Loop

## Overview

Drive an open PR through the agent reviewer (`metal-agent-ops`) and any follow-up rounds until it is approved. Then stop at the merge gate. Reuse `pr-plan-review-comments` and `commit-push-reply-comments`; do not reimplement them.

## When to use

- After shipping PRs, when the user wants the agent reviewer to review.
- When new review findings arrive and must be triaged, fixed, pushed, and replied to.
- For a coordinated frontend + backend repo pair linked to one Linear ticket.

Not for: human-only review. Never merge.

## The loop

Repeat until the reviewer approves or findings stop being valid/in scope.

1. **Request review.** For each PR: mark ready if draft (`gh pr ready <n> --repo <repo>`), then post exactly one trigger:
   ```bash
   gh pr comment <n> --repo <repo> --body "@metal-agent-ops review"
   ```
   Do not double-post for the same round.

2. **Poll every 20 minutes.** Stop when both PRs have a new `metal-agent-ops` review.
   ```bash
   gh api repos/<repo>/pulls/<n>/reviews --paginate \
     --jq '[.[] | select(.user.login=="metal-agent-ops")] | length'
   ```

3. **Re-fetch before triage.** Run `gh pr view <n> --repo <repo> --json headRefOid,updatedAt`. If either changed since the review, re-fetch the threads. Never triage stale data.

4. **Triage.** Use `pr-plan-review-comments`. It fetches unresolved threads, reviews, and `metal-agent-ops` review bodies. Classify each: **Already fixed**, **Still valid**, **Not worth acting on**. Present a numbered list before fixing. Ask the user when validity is ambiguous.

5. **Fix the valid, in-scope findings.** Use `test-driven-development`. Run only the affected tests, never the full suite.

6. **Commit, push, reply.** Use `commit-push-reply-comments`. It owns the reply format and the full-SHA verification. Reply to every thread: the fixing commit, or an explicit "intentional" with the reason. Target zero open threads.

7. **Align the PR body and the Linear ticket** with the shipped behavior. Patch the PR body (`gh pr edit`) when a fix changes the contract. Use `linear` to sync the ticket when the acceptance criteria or contract change. Keep `Resolves ENG-XXX` on the primary PR and `Relates to ENG-XXX` on the companion.

8. **Re-request review** on each changed PR, then loop.

## Decision rules: which findings to fix

| Finding | Verdict |
| --- | --- |
| Critical/High, unmet acceptance criterion, or reproduced correctness bug | Fix |
| Medium, low-risk, mirrors an existing shipped pattern, no scope creep | Fix |
| Contradicts the ticket contract or a deliberate design decision | Push back with the reason and the contract pointer |
| Style nit or bot noise | Not worth acting on |
| Technically questionable, or changes behavior beyond the ticket | Verify first (use `receiving-code-review`); ask if still unclear |

## Guardrails

- Never merge, force-push, or rewrite history.
- Never act on stale review data or on `isResolved` alone; read the current code.
- Do not reply with a short or unverified SHA.
- Work in the ticket's worktrees. Do not disturb a checkout parked on another branch.
- Keep fixes in scope. Out-of-scope findings go to the user, not into the PR.

## Verification before done

The loop is done only when all hold:

- `reviewDecision == APPROVED` at the latest `headRefOid`, for every PR.
- Zero unresolved review threads.
- CI green; no required check failing.
- Every branch-only commit has GPG status `G`.
- PR bodies and the Linear ticket match the shipped behavior.

Then report the PR links and the triage list. Stop at the merge gate; the user merges in the GitHub UI.

## Common mistakes

| Mistake | Fix |
| --- | --- |
| Act on the first review without re-fetching the head | Re-fetch head/updatedAt, then re-fetch threads on change. |
| Treat `isResolved: false` as "unaddressed" | Read the current code and history; classify first. |
| Fix every finding | Triage. Push back on contract conflicts and nits. |
| Reply with a short SHA | Verify the full 40-char SHA from the pushed PR. |
| Forget the companion repo | Request review and reply on both PRs. |
| Merge after approval | Stop at the merge gate. |
