---
name: run-project
description: Use when a Linear project or a set of tickets is ready for delivery and each ticket must run through scope-feature, deliver-ticket, and wrap-up until all tickets reach a terminal state.
---

# Run Project

## Overview

Deliver every ticket in a Linear project, one at a time, until all tickets are terminal. Then write the project post-mortem. This skill owns ordering and gates only. Ticket work delegates to `scope-feature`, `deliver-ticket`, and `wrap-up`.

## When to use

- The user asks to run, deliver, or complete a Linear project.
- A set of related tickets must be worked in dependency order.

`run-project` assumes the project and its tickets already exist. `scope-feature` creates them.

## Progress log

Create the project progress log at `<root>/docs/superpowers/progress/<project-slug>.md`. It has one section per ticket. `scope-feature`, `deliver-ticket`, and `wrap-up` append to the same log. Append at each phase transition.

## Flow

### 1. Load the project

**REQUIRED SUB-SKILL:** Use `linear` to read the project and its tickets, including state, priority, and relations.

### 2. Order the tickets

Order by dependency first, then priority. A blocked ticket waits for its blocker. Present the order and the rationale, then wait for approval before starting.

### 3. Run each ticket

For each ticket, in order:

1. **REQUIRED SUB-SKILL:** Use `scope-feature` for the design and the spec.
2. **REQUIRED SUB-SKILL:** Use `deliver-ticket` from the approved spec through the merge gate.
3. **REQUIRED SUB-SKILL:** Use `wrap-up` after the user merges.

Update the project progress log with the ticket state and the handoff note after each ticket.

### 4. Stop conditions

Stop the run and report when:

- A ticket needs authority outside the approved scope.
- A gate fails and cannot be resolved.
- A dependency is not terminal when its dependent ticket starts.
- The user replaces the request.

Do not skip a ticket or reorder silently. Preserve the project state for resumption.

### 5. Finish the project

When every ticket is terminal:

1. Write the project post-mortem to the retro queue at `~/.agents/retros/<YYYY-MM-DD>-<project-slug>-postmortem.md`.
2. Upload the post-mortem to Linear as a project document or comment through `linear`.
3. Update the progress log with the final state.

## Gates

- The user approves the ticket order before the run starts.
- Each ticket passes its own `scope-feature` and `deliver-ticket` gates. Do not bypass them.
- Never merge. The user merges each PR in the GitHub UI.

## Guardrails

- Do not reimplement `scope-feature`, `deliver-ticket`, or `wrap-up` inline.
- Do not start a ticket whose dependency is not terminal.
- Do not report a ticket as done without its merge gate satisfied.
- Do not expand scope beyond the project.

## Common mistakes

| Mistake | Required behavior |
| --- | --- |
| Running tickets in creation order | Order by dependency, then priority. |
| Starting a blocked ticket | Wait for the blocker to reach a terminal state. |
| Skipping wrap-up | Every ticket gets a post-mortem and a Slack note. |
| Reporting the project done early | Every ticket must be terminal. |
| Merging a ticket PR | Stop at the merge gate. |
