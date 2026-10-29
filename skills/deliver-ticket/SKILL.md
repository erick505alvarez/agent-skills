---
name: deliver-ticket
description: Use when an approved implementation spec exists for one ticket and the work must be built, verified, shipped, reviewed, and brought to the merge gate. Starts at implementation; design and spec come from scope-feature.
---

# Deliver Ticket

## Overview

Build one ticket from an approved spec through to the merge gate. Own sequencing and gates only. Every task delegates to an existing skill by name.

## Progress and friction logs

Create or append the progress log at `<root>/docs/superpowers/progress/<slug>.md` at each phase transition. Append friction entries to `~/.agents/friction/<YYYY-MM-DD>-<task-slug>.md` the moment they happen. Do not defer entries to the end of a phase.

## Resume check

Read the progress log. Check `git worktree list` and the local branch. If a worktree or local checkout already holds the feature branch, resume there — the feature may be partially implemented. Create a worktree only when none exists, through `using-git-worktrees`.

## Model selection

Run implementation under `~deepseek/deepseek-v4-flash-latest` when available. Fall back to any DeepSeek Flash model, then to `gpt-5.6-luna`.

## Flow

### 1. Implement

**REQUIRED SUB-SKILL:** Use `executing-plans` as the execution coordinator and `test-driven-development` — tests first, aiming for 100% coverage of the change.

**REQUIRED SUB-SKILL:** Use `subagent-driven-development` when the spec has multiple tasks. Dispatch one implementer and one reviewer per task. Loop until the reviewer reports no blockers.

**REQUIRED SUB-SKILL:** Use `dispatching-parallel-agents` when the spec has two or more independent tasks. Do not parallelize tasks that edit shared state or consume another task's output. Tell each agent that sibling files may show as pre-existing modifications; verify the combined diff after all agents finish.

Every dispatch states that the spec is approved. The subagent starts TDD from its task brief. It does not reopen design or scoping.

### Subagent completion loop

Keep the ticket active until every implementer, reviewer, and verifier completes. After each dispatch, poll with a 30–60 second wait. Review each result and dispatch the next task immediately. Send the user a concise progress update at least every 60 seconds.

If an agent has no checkpoint for five minutes, interrupt it to preserve work and request a checkpoint: RED/GREEN evidence, changed files, remaining work, and blockers. If it has no useful result after a second five-minute interval, reassign the task to a fresh subagent. Do not treat an agent as complete without a result. A subagent failure starts the diagnosis or fix flow; it does not end the ticket.

### 2. Verify

**REQUIRED SUB-SKILL:** Use `verification-before-completion`.

- Run the spec's scoped test command. All tests pass.
- Check coverage of the changed files. Report the number; return to step 1 if coverage has gaps.
- Report only results backed by real command output.
- Run only targeted tests. Never run a full suite.

For Go tests, set `GOCACHE` and `GOTMPDIR` to writable temp directories and create them with `mkdir -p` first.

### 3. Ship

**REQUIRED SUB-SKILL:** Use `pr-create` to publish the pull request with the Linear ticket linked. `pr-create` chains `commit-push-reply-comments` for dirty work. Mark each PR ready for review with `gh pr ready`.

### 4. Monitor

Watch CI and new reviews until the PR is stable. Report progress at least every 60 seconds. Do not post commentary while a check is still running.

### 5. Fix

**REQUIRED SUB-SKILL:** Use `pr-plan-review-comments` to fetch, triage, and fix valid comments.

Fix valid comments with the same TDD discipline as step 1, and verify per step 2 before pushing. Push through `commit-push-reply-comments`. Every fix push replies to each addressed thread with a verified full-SHA link.

### 6. Merge gate

**REQUIRED SUB-SKILL:** Use `merge-gate`. Verify CI green, no conflict with `main`, no open threads, and approval from a qualifying user. Stop and give the user the PR link. The user merges in the GitHub UI.

## Gates

- The spec is already approved. Do not re-enter design or scoping.
- Commits and pushes happen only through skills whose workflows include them.
- Never merge, force-push, or rewrite history.
- Keep polling until every task, review, fix, and verification completes.

## Guardrails

- Do not reimplement `test-driven-development`, `executing-plans`, `subagent-driven-development`, `dispatching-parallel-agents`, or `verification-before-completion`.
- Do not report test or coverage results without real command output.
- Do not expand scope beyond the spec. Out-of-scope findings go to the user.

## Common mistakes

| Mistake | Required behavior |
| --- | --- |
| Creating a worktree over a partial feature | Run the resume check first. |
| Starting while a reviewer is active | Poll until it returns, then dispatch the follow-up. |
| Treating a slow subagent as complete | Request a checkpoint after five minutes. |
| Full suite as the gate | Run the scoped test command. |
| Pushing a fix without thread replies | Reply to each addressed thread with a verified full-SHA link. |
| Merging | Stop at `merge-gate`; the user merges. |
