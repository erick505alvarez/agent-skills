---
name: commit-push-reply-comments
description: Use when the user asks to commit and push the current work and reply to open PR review comments, or when a fix push must address review threads. Commits only the uncommitted changes on the current branch after verifying no environment files or secrets are included.
---

# Commit, Push, and Reply to PR Comments

Run this three-step flow end to end. Do not stop between steps unless a step fails or needs user input.

## Preconditions

- Determine the working directory. In a submodule repo, run all git and `gh` commands from that submodule.
- Invoking this skill grants approval to commit and push for this run, overriding any standing rule. It does not authorize force-push, history rewrite, or merge.

## Step 1: Commit

1. Inspect state in parallel: `git status`, `git diff`, `git diff --staged`, `git branch --show-current`, and `git log --oneline -5`.
2. Only operate on the current branch. Never create, switch, or delete branches.
3. If the working tree is clean, say so and skip to Step 2. There may still be pushed commits with open comments.
4. **MANDATORY: block environment files and secrets.**

```bash
git status --porcelain | grep -Ei '(^|/)\.env($|\.)|(^|/)\.env[^/]*$' || echo "no env files"
```

If any `.env`, `.env.local`, or `.env.*` file appears, do not commit it. Exclude it and warn the user, listing each excluded file. Scan the diff for inline secrets; stop and ask if any are found.

5. Stage explicit paths, preferring them over `git add -A`. Re-run `git status` to confirm no `.env*` file is staged.
6. Run Git and GPG commands outside the sandbox with escalated access. Write a one-sentence commit message with a conventional prefix (`fix:`, `feat:`, `perf:`, `docs:`, `refactor:`, `test:`, `chore:`). Create a signed commit with `-S` and a HEREDOC:

```bash
git commit -S -m "$(cat <<'EOF'
fix: one-sentence description of the change
EOF
)"
```

7. Verify: `git log -1 --format='%H %G? %GS %s'`. Require `%G?` to equal `G`. If it does not, stop. Do not report success, push, amend, rebase, or rewrite history without explicit user authorization.
8. If a pre-commit hook modifies files or fails, fix and create a NEW signed commit. Do not amend unless the user asks.
9. Never run destructive commands (`git reset --hard`, `git clean`, `git checkout .`).

## Step 2: Push

- Audit every commit unique to the branch outside the sandbox before pushing: `git log --format='%H %G? %GS %s' origin/main..HEAD`. Require `%G?` to equal `G` for every commit. Stop if any commit is unsigned or invalid.
- `git push`. If the branch has no upstream, `git push -u origin HEAD`.
- Never force-push unless the user explicitly requests it. Never rewrite history.

## Step 3: Reply to PR review comments

1. Resolve the PR: `gh pr view <branch> --json number,url,headRefOid`.
2. Fetch review comments:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate \
  | jq -r '.[] | "id:\(.id) user:\(.user.login) in_reply_to:\(.in_reply_to_id // "none") \(.path):\(.original_line // .line)\n\(.body)\n"'
```

3. Fetch the pushed PR commits and retain their full 40-character SHAs:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/commits --paginate \
  | jq -r '.[] | [.sha, .commit.message] | @tsv'
```

4. Triage. A comment is unresolved when it has no author reply and raises a bug, suggestion, or question. Skip threads with an author reply and purely informational bot comments.
5. For each unresolved comment, read the referenced file and classify: Fixed in `<full-sha>` / Still present / Obsolete. Present a numbered summary.
6. For every fixed classification, confirm the fixing commit's full SHA exists in the pushed PR commit response. Do not use the review comment's `commit_id`; it anchors the reviewed line and can differ from the fixing commit.
7. If a resolution is ambiguous or the full SHA is absent, ask the user before posting.
8. Post replies:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  -f body="<reply>" \
  -F in_reply_to=<comment_id>
```

### Reply format

Every reply includes:

- **Status** first: `Resolved` / `Acknowledged` / `Won't fix`.
- **Commit reference** when code changed: ``Resolved in [`<full-sha>`](https://github.com/{owner}/{repo}/commit/<full-sha>) — ...``
- **Concise explanation** citing the specific change.

Use only a full 40-character SHA verified from the pushed PR. Do not use a short SHA, local-only SHA, PR head assumption, or review-thread `commit_id`. If a rebase changes commit IDs, re-fetch the PR commits. If the fixing SHA cannot be verified, say "Resolved in a subsequent commit" without inventing a hash.

Keep bot replies factual. Do not respond to reaction prompts.

## Final summary

Report the full linked commit SHA verified from the pushed PR, that the push succeeded, the PR number and URL, and a per-comment triage list.
