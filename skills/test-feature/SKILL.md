---
name: test-feature
description: Use when the user wants to test a feature by hand in the local checkout. Checks out the feature branch and removes the worktree.
---

# Test Feature

## Overview

Bring a feature branch into the main local checkout so the user can test it by hand, then remove the worktree. This skill is manual and user-invoked.

## Flow

1. Read the progress log at `<root>/docs/superpowers/progress/<slug>.md` for the feature branch, worktree path, and repositories.
2. Confirm the worktree exists. If it does not, check whether the branch is already checked out locally; if it is, skip to step 5.
3. Confirm the worktree has no uncommitted changes:

```bash
git -C <worktree-path> status --short
```

If it reports changes, stop and ask the user how to handle them. Never stash or discard automatically.

4. Remove the worktree. Git requires this before the branch can be checked out in the main checkout:

```bash
git -C <repository> worktree remove <worktree-path>
```

5. Check out the feature branch in the main local checkout:

```bash
git -C <repository> checkout <branch>
```

6. The user tests by hand.
7. Record in the progress log that the worktree was removed.

## Gates

- Do not remove a worktree with uncommitted changes without asking.
- Do not remove the main checkout.
- Do not push, commit, or merge. This skill changes the local checkout only.

## Running the app locally

The main checkout already has `node_modules`. Start the frontend with `make dev`, and confirm the backend stack is running.

For visual QA of the checked-out work, use `visual-qa`. The per-port CORS allow-list and worktree `node_modules` guidance also lives in `visual-qa`.

After you change backend Go code, `metallica-api` hot-reloads through `air`. A silent `air` build failure leaves the old binary serving and the UI shows stale behavior. Confirm the running binary is newer than the change, or restart the service:

```bash
docker-compose restart metallica-api
docker exec backend-metallica-api-1 stat -c %y /app/service/tmp/main
```

## Common mistakes

| Mistake | Required behavior |
| --- | --- |
| Checking out before removing the worktree | Git rejects a branch checked out in two worktrees. Remove first. |
| Removing a worktree with uncommitted changes | Stop and ask. |
| Assuming the worktree exists | Read the progress log first. |
| Forgetting to record the removal | Update the progress log so `wrap-up` skips removal. |
