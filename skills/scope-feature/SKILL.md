---
name: scope-feature
description: Use when the user describes a feature, concept, or bug fix and wants it scoped into a design and an implementation spec. Decides whether the work is one ticket, several tickets, or a Linear project. For building an approved spec, use deliver-ticket. For a project of tickets, use run-project.
---

# Scope Feature

## Overview

Turn a request into an approved implementation spec. Own the scale decision, the definition-of-ready gate, codebase reconnaissance, the design, and the spec. This skill owns sequencing and human gates only. Every task delegates to an existing skill by name.

## When to use

- The user describes a feature, bug, or chore and wants it designed and planned.
- The user shares a Linear ticket and wants it prepared for development.

For a read-only ticket brief, use `linear`. For building an approved spec, use `deliver-ticket`. For a project of tickets, use `run-project`.

## Progress log

Create or append the progress log at `<root>/docs/superpowers/progress/<slug>.md`. Record scope and identifiers, the artifact index, progress, findings, the friction-log path, and the handoff note. Append at each phase transition.

## Flow

### 1. Scale decision

Decide the shape before any design work:

- **Single ticket** — continue through design and spec for that ticket.
- **Multiple tickets** — decompose into tickets with unambiguous acceptance criteria, declared file scope, and dependencies. Create them through `linear`. Then hand each ticket to `deliver-ticket`.
- **Linear project** — decompose into a project with tickets. Create the project and tickets through `linear`. Stop. The user starts `run-project`.

Present the breakdown and wait for approval before any Linear write.

Re-check the shape after reconnaissance: if the work needs new backend behavior or a new interface rather than a frontend consumer of an existing one, upgrade the classification and require a written spec.

### 2. Intake

**REQUIRED SUB-SKILL:** Use `linear` to fetch the issue, comments, project context, relations, and linked PRs or branches.

Extract the ticket type (bug / feature / chore / spike), acceptance criteria or reproduction steps, and priority. If the ticket bundles multiple independent problems, propose splitting it.

### 3. Definition-of-ready gate

Confirm the request has enough to act on. Bugs need a clear failure description; known triggering conditions, affected users, or log references are enough for non-deterministic bugs. Features need acceptance criteria or a clear description of desired behavior.

If anything is missing or ambiguous, list every gap, then ask one question per message, highest-impact first. Do not proceed on assumptions.

### 4. Codebase reconnaissance

Identify the affected repositories. Dispatch subagents to:

- Find the relevant files, modules, and tests.
- Read the applicable `AGENTS.md`/`CLAUDE.md` rules and any `.cursor/rules`.
- Search for existing patterns used for similar problems (`git log -S`, `git log --grep`).
- Identify blast radius and the user-permission checks for any data entities involved.
- For any user-facing value set, name the accepted vocabulary and the source of its documentation as two separate decisions.
- Before proposing a pattern, confirm a call site already uses it; a pattern with no existing call site is a new pattern and needs justification.
- When a design reuses an existing endpoint or path, confirm it has the semantics the feature needs — reusing a route is not reusing its behavior. For example, verify that a bulk update writes back to the underlying record, not only the list cell.
- Enumerate every concrete implementation of each interface or transport you will extend (for example every configured mail provider). One example does not represent the set.
- Confirm the visibility of each symbol you plan to reuse; a package-private helper needs a new exported or isolated helper before the design can depend on it.
- Verify every file path named in the ticket or its comments; a ticket can
  cite a stale or wrong path.
- When the ticket or a developer note names a cross-cutting project or
  initiative, read that project's scope before adopting its guidance. Do not
  apply guidance that the project's scope excludes.

### 5. Duplicate-work check

Per affected repo: `git log --all --grep=<ticket-id>`, `git branch -a`, and `gh pr list --search <ticket-id>`. Surface any existing work and ask whether to continue it.

When several checkouts point at the same remotes, dedupe the search by remote — one search per unique remote is enough.

Also search Linear for an existing broader spec or general ticket that this work is a specific application of, and link it as related.

### 6. Root cause (bugs only)

If the root cause is not already established, **REQUIRED SUB-SKILL:** Use `systematic-debugging` through Phase 3 (Hypothesis and Testing) only. Stop before Phase 4. If the bug cannot be reproduced locally, look for logs, traces, or a session recording before guessing.

### 7. Best-practice research

Research how this class of problem is solved outside this codebase. Keep it scoped to the specific problem.

### 8. Surface conflicts

Compare best-practice research against local patterns. For every conflict, state both options, recommend one with a one-line rationale, and flag any approach that duplicates existing logic.

### 9. Design

**REQUIRED SUB-SKILL:** Use `brainstorming` with the user to develop the approach. Ground it in step 4: reuse and extend existing patterns; justify a new abstraction. Enumerate edge cases: error, empty, loading, permissions and tenancy, concurrency, and failure modes.

If the work includes frontend UI, use the brainstorming visual companion to create mockups. Iterate until the user approves the design.

### 10. Spec

**REQUIRED SUB-SKILL:** Use `writing-plans`. The spec must include:

- TDD steps for every task.
- A coverage target: all new and changed code covered by tests.
- The exact scoped test command for this change.
- A mapping of every acceptance criterion to the task that satisfies it; flag any criterion with no covering task.
- A rollout and verification section when the change ships to users.
- A definition-of-done checklist: tests, lint, type-check, docs, and ticket updates.
- The design spec and decisions live under the root's `docs/superpowers/specs/`; the implementation plan lives under the root's `docs/superpowers/plans/`, not a submodule's `docs/`.

Record the friction-log path in the spec. Create the session friction log at `~/.agents/friction/<YYYY-MM-DD>-<task-slug>.md`.

### 11. Plan review

Dispatch one independent reviewer for the spec. Give it the spec path and the plan-derived file list. The reviewer checks task boundaries, dependencies, invariants, test coverage, rollback behavior, conflict with repository rules, and that every named fixture, mock, helper, and command exists. It must not edit files.

Update the spec for every valid finding. Record a rationale for each rejected finding that affects implementation. The revised spec is the only spec `deliver-ticket` receives.

### 12. Gate and branch

The user approves the spec. No code changes before approval. Attach approved mockups to the ticket; verify the attachment succeeded.

Create the ticket branch per affected repo:

1. `git status --porcelain` — if dirty, stop and ask.
2. `git branch --show-current` — if not `main`, stop and ask.
3. If clean and on `main`: pull latest, then create the branch.

Branch name: `<username>/<ticket-id-lowercase>-<kebab-slug>`. Match an established repo convention when one exists.

Update the progress log with the spec path and branch.

## Gates

- The user approves the design and the spec. Never skip these.
- Never write to Linear before breakdown approval.
- No code changes before spec approval.
- Never commit files under `docs/superpowers/`.

## Guardrails

- Do not reimplement `linear`, `brainstorming`, `writing-plans`, or `systematic-debugging` inline.
- Never invent acceptance criteria or reproduction steps.
- Never silently resolve a conflict.
- Never switch branches or discard changes in a dirty repo without asking.
- Do not claim mockups are attached without verifying.

## Common mistakes

| Mistake | Required behavior |
| --- | --- |
| Designing before reading the code | Complete reconnaissance first. |
| One mega-ticket | Decompose until each ticket is one `deliver-ticket` run. |
| Silently dropping acceptance criteria | Map every criterion or flag the gap. |
| Coding before spec approval | The spec gate is hard. |
| Skipping the plan reviewer | Dispatch one independent reviewer before approval. |
| Committing the spec | Files under `docs/superpowers/` stay local. |
