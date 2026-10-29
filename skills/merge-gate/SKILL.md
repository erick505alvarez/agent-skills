---
name: merge-gate
description: Use when a pull request is ready to check the merge gates: CI green, no conflict with main, all review threads resolved, and approval from a qualifying user. Stops and hands the merge to the user.
---

# Merge Gate

## Overview

Verify the four merge gates on a pull request. Report the result. Stop. The user merges in the GitHub UI. This skill never merges.

## Gates

| Gate | Passing condition |
| --- | --- |
| CI | Every required check passes. |
| No conflict | The PR merges cleanly into `main`. |
| Comments addressed | No unresolved review threads and no unaddressed actionable review bodies. |
| Approval | The PR has an approving review from a qualifying user. |

## Checks

Resolve the PR first: `gh pr view --json number,url,headRefName`. In a monorepo with submodules, run `gh` from the submodule that owns the PR.

### 1. CI

```bash
gh pr checks <pr> --required
```

Every required check must pass. A pending check is not passing; wait and re-check.

### 2. Conflict with main

```bash
gh pr view <pr> --json mergeable,mergeStateStatus
```

Require `mergeable` to be `MERGEABLE`. A `CONFLICTING` value fails the gate. A `UNKNOWN` value means GitHub is still computing; wait and re-check.

### 3. Comments addressed

```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes { isResolved }
      }
    }
  }
}' -f owner=<owner> -f repo=<repo> -F pr=<number>
```

Every thread must be `isResolved: true`. Also confirm the latest `metal-agent-ops` review body has no unaddressed actionable finding, per `pr-plan-review-comments`.

### 4. Approval

```bash
gh pr view <pr> --json reviewDecision,reviews
```

Require `reviewDecision` to be `APPROVED`. A qualifying user is a non-bot reviewer whose latest review is an approval. If the repository enforces branch protection, GitHub computes `reviewDecision` from it; trust that value.

## Report and hand off

Report each gate with its command and result. If every gate passes, give the user the PR URL and stop:

> All merge gates pass. Merge in the GitHub UI: <PR URL>

If a gate fails, report the failing gate, the evidence, and the next action. Do not merge.

## Guardrails

- Never merge. The user merges in the GitHub UI.
- Never treat a pending check, an `UNKNOWN` mergeable state, or an unresolved thread as passing.
- Never approve on the user's behalf.
- Report only results backed by real command output.
