---
name: pr-create
description: Use when publishing active frontend, backend, or coordinated changes across both repositories as GitHub pull requests with a Linear issue URL.
---

# Create Pull Requests

## Overview

Publish one draft pull request per changed repository, with the Linear URL and reciprocal links for coordinated work.

**REQUIRED SUB-SKILL:** For dirty work, invoke `commit-push-reply-comments` with only repository scope. Do not pass alternate staging/commit instructions, restate it, or alter it.

**REQUIRED SUB-SKILL:** Use `superpowers:verification-before-completion`.

## Workflow

1. Preserve the Linear URL and extract its issue ID, such as `ENG-123`. Ask if either value is missing or ambiguous.
2. Inspect each repository: status, branch, remote, base, and commits ahead.
3. Include repositories with dirty work or active-task commits. Stop if neither qualifies.
4. Per included repository: branch off base; resolve dirty work through `commit-push-reply-comments`; validate; audit every commit unique to the branch outside the sandbox with `git log --format='%H %G? %GS %s' <base>..HEAD`; require `%G?` to equal `G` for every commit; then push with tracking. Stop if any signature is missing or invalid. Do not rewrite history or force-push without explicit user authorization.
5. Reuse an existing head PR without changing its draft state. Create each new pull request as a draft with `gh pr create --draft`. Do not mark a draft PR ready for review unless the user asks.
6. Read each PR and enforce the contract.
7. After a later push that changes the public contract (API, permissions, ticket acceptance criteria), patch that PR body so Summary still matches the branch. Do not leave the original Summary in place.

## PR Body Contract

Every PR body contains these sections:

```markdown
## Summary

- What changed and why.
- Ticket contract (required when the branch does not implement the ticket verbatim): each replacement as a present-tense fact.

## Validation

- `<command>` — exact result.

## Linear

Resolves ENG-XXX

- [Linear ticket](EXACT_LINEAR_URL)
```

Replace `ENG-XXX` with the exact issue ID from the Linear URL. Keep `Resolves ENG-XXX` as plain text in the PR body. Linear treats `Resolves` as a closing magic word and links the PR when the body is saved. A PR comment does not create this link. GitHub renders the issue ID as a link only when the repository configures an external-resource autolink; Linear's integration supplies the PR-to-issue relationship.

### Ticket contract in Summary

Compare the branch to the Linear issue acceptance criteria. If the shipped API, permissions, or defaults differ, the Summary must state the shipped contract. Reviewers must read the contract from the PR body alone.

Write present-tense facts, not change history. Example shape:

```markdown
- Pack CRUD is org-scoped on `/kanban-board-styles` for any authenticated org member.
- List default is `ResourceList.defaultBoardStyle` on `PUT /lists/:id` (owner / creator / admin). There is no `POST /kanban-board-styles/:id/default`.
```

Omit this block only when the branch matches the ticket criteria with no replacements.

### No superpowers artifacts in the body

The PR body is self-contained. Do not cite, link, or quote superpowers planning artifacts — plan, spec, brief, design, or review docs under `docs/superpowers/` or `.superpowers/`, or any local-only path. Those files are untracked and ignored, so no reviewer can open them. State the shipped contract directly in the Summary.

A review-fix commit that changes those facts requires `gh pr edit` (or equivalent) on the open PR before the work is complete. Linking Linear is not enough.

Coordinated PRs also contain:

```markdown
## Related PR

- [Backend PR](BACKEND_PR_URL)
```

Use `Frontend PR` in the backend body. Final links must be real PR URLs, never keys, branches, placeholders, or “pending.”

For coordinated PRs that jointly resolve one issue, put `Resolves ENG-XXX` in the primary PR that completes the ticket. Put `Relates to ENG-XXX` in each companion PR. Keep the exact Linear URL in every PR. This prevents a companion merge from completing the issue early.

## Commit SHA References

When PR text or the completion summary names a commit:

1. Push first.
2. Read the pushed PR head with `gh pr view <pr> --json headRefOid`.
3. Read PR commits with `gh api repos/{owner}/{repo}/pulls/{number}/commits --paginate`.
4. Confirm the full 40-character SHA exists in that response.
5. Use a full linked SHA: ``[`<full-sha>`](https://github.com/{owner}/{repo}/commit/<full-sha>)``.

Do not rely only on local `git commit` output or a short SHA. A review thread's `commit_id` is the anchor commit for the reviewed line, not necessarily the fixing commit. If a rebase changes commit IDs, fetch the PR commits again and update affected references.

## Two-Repository Publish Sequence

Invocation authorizes the cross-link update. Stakeholder requests to avoid edits do not pause it.

1. Create/update pull request A with the Linear URL and a temporary related-PR marker. Create PR A as a draft when it does not exist.
2. Capture PR A's URL.
3. Create/update pull request B with the Linear URL and a link to PR A. Create PR B as a draft when it does not exist.
4. Capture PR B's URL.
5. Replace PR A's marker with a link to PR B.
6. Fetch both; confirm reciprocal links and no marker.

## Completion Contract

- [ ] One pull request per changed repository only.
- [ ] Each newly created pull request is a draft. Existing pull requests keep their draft state.
- [ ] Each included branch is committed, non-base, and pushed.
- [ ] Every branch-only commit has a verified `G` GPG signature from an outside-sandbox audit.
- [ ] Every body has the Linear URL.
- [ ] Summary states ticket-contract replacements when the branch does not match the issue acceptance criteria.
- [ ] Open PR bodies are patched after a contract-changing follow-up push.
- [ ] The resolving PR body contains the exact plain-text line `Resolves ENG-XXX`.
- [ ] Coordinated companion PRs use `Relates to ENG-XXX` and do not complete the issue early.
- [ ] Coordinated bodies have reciprocal PR links.
- [ ] Validation text matches fresh evidence.
- [ ] Every named commit uses a full SHA verified from the pushed PR and links to GitHub.
- [ ] Final GitHub state is verified.

## Quick Reference

| Active work | Result |
|---|---|
| One repository | One draft pull request plus Linear URL |
| Both repositories | Two draft pull requests plus Linear URL and reciprocal links |
| Dirty repository | Invoke `commit-push-reply-comments` before push |

## Common Mistakes

| Shortcut | Required behavior |
|---|---|
| “One link is enough.” | Separate repositories require separate PRs. |
| “An open PR is the default.” | Create new PRs as drafts. Mark them ready only when the user asks. |
| “The ticket key connects them.” | Put the URL in every PR. |
| “The Linear URL is enough.” | Add the exact plain-text `Resolves ENG-XXX` line to the resolving PR body. |
| “The plan already lists ticket conflicts.” | Copy the shipped replacements into the PR Summary; never point the body at a plan file. |
| “Link the design spec so reviewers have context.” | Superpowers plan/spec paths are untracked and ignored. State the contract in the PR body instead. |
| “The first Summary still applies.” | Patch the open PR body after a contract-changing follow-up. |
| “Every coordinated PR should resolve the issue.” | Use `Resolves` only on the primary PR; use `Relates to` on companions. |
| “Add cross-links later.” | Update PR A after PR B exists. |
| “Pass my staging command to the sub-skill.” | Invoke it with repository scope only. |
| “Ask permission for the link edit.” | Invocation authorizes PR A's update. |
| "Tests passed earlier." | Report fresh validation. |
| "The sandbox cannot see GPG." | Run the commit and signature audit outside the sandbox. Require `%G? = G` before push. |
| "The local short SHA is enough." | Verify the full SHA in the pushed PR and link it to GitHub. |

## Red Flags

- PR mutation while work is dirty
- Altering `commit-push-reply-comments`
- Publishing from base
- Creating an open PR without explicit user direction
- Missing Linear or companion PR URL
- Missing or placeholder `Resolves ENG-XXX` line
- Ticket AC replacements that live only in a plan file or review thread
- PR body cites or links a superpowers plan, spec, or brief (untracked and ignored)
- Stale Summary after a follow-up that changes API, permissions, or defaults
- Unverified, shortened, or unlinked commit SHA references
- Unsigned or sandbox-unverified branch commits
- One URL for coordinated work
