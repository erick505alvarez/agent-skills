---
name: maintaining-docs
description: Create and maintain feature documentation under a project's root-level `docs/` folder (e.g. `frontend/docs/`, `backend/docs/`, or a repo's top-level `docs/`). Use when adding, updating, restructuring, or reviewing markdown in a `docs/` folder, when splitting a large doc into cross-referencing topic files, or when keeping those docs in sync with code changes. Enforces succinct, pointer-based writing that captures gotchas and edge cases without polluting agent context.
---

# Docs Maintenance

Feature docs in a project's root-level `docs/` folder are read into an agent's context when it works on that feature. Every line competes for context space, so be ruthless: capture the insights and gotchas an agent could not infer from the code, and nothing else.

`docs/` here means the docs folder at a repo or submodule root — e.g. `frontend/docs/`, `backend/docs/`, or a top-level `docs/`. Pick the one nearest the code being documented.

## When to use

- Documenting a new feature, or a change that alters contracts/behavior an agent must know.
- Restructuring one large doc into a feature folder of focused, cross-referencing files.
- Keeping existing `docs/` files in sync after code changes.

## Structure

- One folder per feature: `<root>/docs/<feature>/`.
- `index.md` is the hub: a one-paragraph overview, a **documentation map** linking every topic file with a one-line summary, key source entry points, and the cross-cutting invariants that apply everywhere.
- Topic files cover one area each (e.g. `api.md`, `rendering.md`, `editing.md`). Split when a file mixes unrelated concerns or grows long enough to crowd context.
- Every file starts with a one-line parent link back to `<root>/docs/<feature>/index.md` and cross-links sibling topics where they connect.

## Reference style (paths matter)

- Use **full repo-relative paths** in all references: `frontend/src/app/.../page.tsx`, `backend/services/.../controller.go`, `<root>/docs/<feature>/api.md`. Docs live outside the source tree, so relative or bare-filename references are ambiguous.
- Point, don't copy. Reference file paths, function names, and symbols instead of pasting code — snippets go stale. Use a `path` or `path:line` pointer to the source of truth.
- When a feature crosses a service or API boundary, reference both sides: e.g. a frontend doc points at the backend route registration and handler; a backend doc points at the worker/handler that owns side effects and any consuming client.
- Add a top-of-file pointer comment in the feature's main source entry points directing agents to the index (e.g. `// Feature docs: see <root>/docs/<feature>/index.md`). Keep it after any required directive such as `"use client"` or a package clause.

## Writing rules

- Be succinct. Each line must pass: "Would removing this make an agent more likely to err?" If not, cut it.
- Prioritize **gotchas, edge cases, and corner cases**: non-obvious coupling, files that must change together, intentional-looking-like-bugs, ordering/invariants, fallback paths, and inputs that break naive assumptions.
- Skip anything the code already states plainly, generic engineering advice, and code-style rules.
- Use `IMPORTANT:` sparingly — only for things that cause real breakage if missed.
- Present-tense, state-based language. No temporal/changelog framing (no "now", "previously", "used to", "legacy") unless documenting a genuine migration or deprecation.

## Capturing gotchas

Gotchas are the highest-value content. Find them deliberately, don't wait to stumble on them:

- **Mine the diff and history.** Read the actual change (see "Diff against main"), and `git log -p`/`git blame` on the touched lines — commit messages and PR context often state the *why* and the trap being worked around.
- **Hunt the non-obvious.** Look for: things that look like bugs but are intentional; files that must change together; ordering/timing requirements; invariants not enforced by types; fallback/legacy paths; inputs that break the naive case; cross-service coupling; and silent-failure modes.
- **Follow the fallbacks and error paths.** Most gotchas hide in `else`, `catch`, defaulting, and back-compat branches — note what triggers them and what an agent breaks by ignoring them.
- **Confirm, don't guess.** Verify a suspected gotcha against the code before writing it; a wrong gotcha is worse than none.
- **Write each as: trap → why → what to do.** State the surprising behavior, the reason it exists, and the action an agent must take. Anchor it to a concrete `path`/symbol. One tight bullet beats a paragraph.
- **Prune fixed ones.** If a change removes a gotcha, delete its doc entry rather than leaving a historical note.

## Diff against main (be thorough)

Base documentation on what actually changed, not assumptions. Use the full branch diff, not just the latest commit:

- `git diff main...HEAD --name-only` — every file changed since the branch diverged (note `...`, the merge-base diff).
- `git diff main...HEAD -- <dir>` — read the real diffs per feature directory; understand new/changed/removed contracts, behavior, side effects, and defaults.
- `git log main..HEAD --oneline` then `git log -p main..HEAD -- <path>` — commit-level intent and the reasoning behind tricky changes.
- For renames/moves, `git diff main...HEAD --stat -M` so doc references and cross-links track the new paths.
- Cross the boundary: when a frontend change has a backend counterpart (or vice versa), diff both sides so the cross-references and endpoint pointers stay accurate.

## Process

1. **Scope it.** Diff the branch against `main` (see "Diff against main") and group changes by feature; only touch docs for areas with meaningful (non-formatting) changes. For new docs, identify the feature's entry points and boundaries, and the nearest root-level `docs/` folder.
2. **Choose structure.** Single file for a small feature; a folder with `index.md` + topic files once it spans multiple concerns.
3. **Write/update.** Apply the reference style, writing rules, and "Capturing gotchas". Keep the `index.md` documentation map and all cross-links accurate when files are added, renamed, or removed.
4. **Sync pointers.** Ensure the feature's source entry points carry the top-of-file pointer comment, and that every doc cross-reference resolves.
5. **Verify.** Re-read each changed file and confirm: no stale links or renamed-file references, no orphaned topic file missing from the index, full paths used, cross-boundary references included where relevant, and nothing redundant with the code.
