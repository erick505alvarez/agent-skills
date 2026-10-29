---
name: pr-code-review
description: Use when reviewing GitHub pull requests, PR diffs, code changes, Linear-linked implementation work, frontend changes, backend changes, or requests for manual code review without CodeRabbit.
---

# PR Code Review

## Overview

Review the PR against its intended behavior, then review the code for defects.
Do manual review. Do not use CodeRabbit or `coderabbit` commands.

## Workflow

1. Gather context:
   - Confirm the repo and PR number or branch.
   - Fetch PR metadata with `gh pr view`.
   - Fetch the full diff with `gh pr diff`.
   - Fetch CI results with `gh pr checks`.
   - Fetch existing review comments to avoid duplicates.
   - Fetch the linked Linear ticket from the PR title, body, branch, or commits.

2. State intent:
   - Summarize the Linear ticket acceptance criteria.
   - Summarize what the PR appears to change.
   - Note any gap between ticket intent and PR scope.

3. Review the diff:
   - Read changed files and nearby code.
   - Trace changed API contracts across callers and consumers.
   - Review tests for the changed behavior and use CI results for execution status.
   - Do not run tests locally. The CI/CD pipeline runs the test suite.
   - Search for gotchas as a required review phase.

4. Search for gotchas:
   - Always read `references/gotchas.md`.
   - Read `references/frontend.md` when the diff touches `frontend/`.
   - Read `references/backend.md` when the diff touches `backend/`.
   - Use `rg` for related symbols, route names, hooks, models, events, and flags.

5. Return findings:
   - Lead with actionable issues, ordered by severity.
   - Include file path, line, impact, and concrete fix.
   - Put uncertainty in open questions, not findings.
   - Do not post to GitHub unless the user explicitly asks.

## Severity

| Severity | Use for |
| --- | --- |
| Critical | Data loss, security bypass, cross-tenant access |
| High | Common-path runtime failure, major incorrect behavior, broken API contract |
| Medium | Proven edge-case bug, supported-platform compatibility bug, bounded acceptance-criteria failure, missing cleanup, risky missing test |
| Low | Non-user-visible maintainability or supportability issue with clear future cost |

Skip style-only comments unless they hide correctness or support risk.

Choose severity from user impact, scope, and likelihood—not merely whether the
behavior conflicts with ticket wording. An unmet acceptance criterion is High
when core or common-path behavior is substantially broken; a failure limited to
specific supported conditions is Medium. A proven user-visible behavior bug is
not Low.

## Review Decision

Severity and the GitHub review event are related but separate decisions:

| Findings | Default event |
| --- | --- |
| Any Critical or High | `REQUEST_CHANGES` |
| Medium that must be fixed before merge because it violates an explicit acceptance criterion, supported-platform contract, or user-correctness requirement | `REQUEST_CHANGES` |
| Medium that is safe to defer | `COMMENT` |
| Low only | `COMMENT` |
| No actionable findings | `APPROVE` |

If recommending `REQUEST_CHANGES`, identify the merge-blocking impact. A Low
finding alone does not justify blocking approval; if it must block, reassess its
severity against the table above.

## Output

```markdown
## Summary

[What changed and whether it matches the Linear ticket.]

## Findings

1. **[Severity] [Title]**
   - Location: `path/to/file.ext:123`
   - Impact: [User, data, security, or operational effect.]
   - Fix: [Concrete change.]

## Open Questions

- [Only include if an answer can change the review result.]

## Verification

- Reviewed Linear ticket: [ticket key/link or "not found"]
- Reviewed diff: [PR number/branch]
- Gotcha searches: [symbols, paths, or "not completed" with reason]
- CI checks: [results or "not available" with reason]

## Recommendation

**APPROVE** | **COMMENT** | **REQUEST CHANGES**
```

If there are no findings, write `No actionable findings found.` under
`## Findings`. Do not invent issues.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Reviewing only the changed lines | Read nearby code and trace callers. |
| Trusting the PR title | Compare behavior to the Linear ticket. |
| Skipping gotcha searches | Run `rg` during review. |
| Running tests locally | Use the CI/CD pipeline results from `gh pr checks`. |
| Mixing questions with findings | Use findings only for proven risks. |
| Using CodeRabbit | Do the manual workflow in this skill. |
