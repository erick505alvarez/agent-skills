---
name: wrap-up
description: Use when a delivered Linear ticket or project is complete and the ticket text, progress log, friction log, post-mortem, and Slack note need to be finalized.
---

# Wrap-up

## Overview

Close the loop on a completed ticket or project: align the Linear issue with what shipped, review the friction log, write the post-mortem, post the Slack note, and clean up local artifacts. Linear writes go through `linear`.

## Workflow

### 1. Collect completed-work facts

Gather from the spec, PRs, progress log, and final verification output:

- Ticket or project identifier
- Repositories touched
- Final PR URLs and numbers
- Final status of each acceptance criterion
- Test and coverage results actually run
- Scope changes, deviations, and follow-ups
- Feature worktree path, if one exists

Use only facts with real command output or approved artifacts. If a fact is missing, ask once before guessing.

### 2. Update the Linear ticket

**REQUIRED SUB-SKILL:** Use `linear`.

- Prefer an issue comment for additive content: summary, PR links, verification, scope notes.
- Update the description only when the ticket text is stale.
- Do not change state, labels, priority, assignee, or project unless the user asks.

Comment body:

```
## Wrap-up

- Outcome: <what shipped>
- PR: <full GitHub PR URL>
- Verification: <exact scoped command and result>
- Notes: <deviations, follow-ups, coverage number>
```

Read the issue again and confirm changed and preserved fields.

### 3. Review the friction log

Read `~/.agents/friction/<YYYY-MM-DD>-<task-slug>.md`. If the path is not recorded, list `~/.agents/friction/` and pick the log whose slug and date match. If no log exists, skip this step.

1. Group entries by the skill they implicate.
2. For each group, propose a concrete edit: target file, section, before/after text.
3. Present the proposals and wait for approval.
4. Apply approved edits, then move the log to `~/.agents/friction/done/`.

Only propose fixes that generalize to future tasks. One-off quirks do not belong in skills.

### 4. Clean up local artifacts

Read the progress log at `<root>/docs/superpowers/progress/<slug>.md`. Delete the specs, plans, and decisions it lists under `<root>/docs/superpowers`, and the archived friction logs under `~/.agents/friction/done/`. Keep the progress log as the final handoff record.

Never delete a file outside `<root>/docs/superpowers` or `~/.agents/friction/done/`.

### 5. Write the post-mortem

Write to the retro queue:

- Ticket: `~/.agents/retros/<YYYY-MM-DD>-<TICKET-ID>-postmortem.md`
- Project: `~/.agents/retros/<YYYY-MM-DD>-<project-slug>-postmortem.md`

Body: outcome, PR links, gates passed, friction themes, skill edits applied, follow-ups.

### 6. Post the Slack release note

Read `SLACK_WEBHOOK_URL` from the environment and post through `curl`. If unset, print the message and ask the user to post it.

Body: ticket ID and title, one-line outcome, PR link, Linear link.

### 7. Remove the worktree

Remove the feature worktree only if `test-feature` did not remove it, per the progress log.

1. Confirm the exact path appears in `git -C <repository> worktree list --porcelain`.
2. Confirm the path is not the main checkout.
3. Run `git -C <worktree-path> status --short`. If it reports changes, stop and ask the user.
4. Run `git -C <repository> worktree remove <worktree-path>`.
5. Confirm the path is absent from `git -C <repository> worktree list --porcelain`.

Keep the branch because the PR can need follow-up commits.

## Gates

- Ticket edits happen only through `linear`, after reading the issue.
- Friction-log skill edits are proposed and approved before any skill file changes.
- Cleanup touches only the paths in step 4.
- Remove only the linked feature worktree.

## Guardrails

- Do not invent PR links, test results, or outcome claims.
- Do not bulk-rewrite ticket fields; send only the fields being aligned.
- Do not archive an unprocessed friction log.
- Do not remove the main checkout or a worktree with uncommitted changes.

## Common mistakes

| Mistake | Required behavior |
| --- | --- |
| Sending every issue field to preserve it | Read first; send only changed fields. |
| Moving the ticket to Done automatically | Only change state when the user asks. |
| Applying one-off task quirks to skills | Keep only changes that generalize. |
| Deleting the progress log | Keep it as the final handoff record. |
| Removing a worktree by directory name alone | Confirm it is the linked feature worktree. |
