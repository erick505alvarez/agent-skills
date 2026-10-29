---
name: review-fix-findings
description: Use when a code review, PR, or bot report leaves multiple findings and each one needs a decision before code changes — especially when the user asks to walk through, discuss, or step through the findings.
---

# Review Findings

A review with several findings is a set of decisions, not a patch list. Verify each finding against the current code, walk the user through it, and change code only after the user decides. One finding at a time.

**Core principle:** No code change before a per-finding decision.

## When to use

- A PR, code review, or bot report lists multiple findings.
- The user asks to walk through, discuss, or step through findings.
- Findings mix valid issues, false positives, and deliberate decisions.

When not to use: the review holds one unambiguous finding (just fix it), or the user explicitly declines a per-finding review ("fix them all, skip the walkthrough"). "Address the findings" is not that.

## The loop

Repeat for each finding, in order.

### 1. Verify against the current code

Read the code at the finding's path today. Do not trust the reviewer's snippet or line number. Classify:

- **Already fixed** — a later commit changed it.
- **Still valid** — it reproduces today.
- **Not worth acting on** — nit, bot noise, or a deliberate decision.
- **Defer** — valid but out of scope.

For PR findings, fetch threads and bodies with `pr-plan-review-comments` and keep the thread URL.

### 2. Present the finding, then wait

One finding per message, in this shape:

```
### Finding N: <title> (<kind>)
Context  — what the code does today and why the finding exists.
Example  — the concrete input or flow where it bites. Name the value.
Fix      — the proposed change and what it affects.
Decision — I suggest <fix / skip / defer>. Proceed?
```

Wait for the decision before touching code.

### 3. Apply only after approval

- Approved: write a failing test first (`test-driven-development`), confirm it fails without the fix, apply the fix, confirm it passes. Run the targeted tests and the build.
- Skipped or deferred: record it where it survives — PR body, ticket, or a deferred list.

Commit, push, and reply through `commit-push-reply-comments`.

### 4. Next finding

Do not fold later findings into the same change.

## Test-related findings

Test-related findings are mandatory. Always fix them manually, in the code, before other findings.

- A test-related finding is any finding about tests, test data, fixtures, mocks, assertions, coverage, or test setup.
- Do not skip, defer, or dismiss a test-related finding. The standing decision is fix.
- Fix the code by hand. Do not rely on a bot, a generated patch, or an automated fixer.
- Fix test-related findings first, then present the remaining findings one at a time.

## Rules

- One finding per decision. No batch edits.
- Test-related findings are always fixed manually. See Test-related findings.
- "Address the findings", "do the work", or an autonomous run is not a decision. Verify and present every finding, then stop.
- A behavior-preserving refactor is still a finding: present it and wait.
- Verify before agreeing. Refute a false or already-fixed finding with evidence.
- Never change behavior the user chose deliberately. Surface conflicts and ask.
- A nit still needs a decision.
- The user may narrow the set ("just fix 1 and 3"). Do exactly that.

## Common mistakes

| Mistake | Fix |
| --- | --- |
| Fixing every finding in one pass | One finding, one decision, one change. |
| Trusting the reviewer's snippet | Read the current code first. |
| Performative agreement | Verify; refute false findings with evidence. |
| Editing before the user decides | Present Context / Example / Fix, then wait. |
| Silently dropping a finding | Record it in the PR body, ticket, or deferred list. |

## Red flags — STOP

- You are about to edit more than one finding's files.
- You are about to edit an "unambiguous" or "behavior-preserving" finding with no decision on the record.
- You disagreed with a finding but are implementing it anyway.

## Related

- `pr-plan-review-comments` — fetch and triage PR threads and review bodies.
- `test-driven-development` — failing test first for each approved fix.
- `verification-before-completion` — evidence before claiming a fix works.
- `commit-push-reply-comments` — commit, push, and reply.
