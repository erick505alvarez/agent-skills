---
name: tool-documentation
description: Use when adding, changing, or reviewing agent tool schemas, field descriptions, defaults, permissions, examples, or tool-facing API contracts.
---

# Tool Documentation

## Overview

Ground each claim in the repository contract. Document agent choices, platform rules, and omission behavior.

## Workflow

1. Read the nearest repository instructions.
2. Select the target contract:
   - For existing-tool documentation, executable behavior is the target.
   - For feature scoping, the approved user or ticket contract is the target. Code supplies patterns and gaps.
3. Trace registration, schema, typed input, execution, adapter request, API model, controller, service, and tests.
4. Sample the target tool and two close same-domain tools or fields.
5. Write the tool-level and field-level documentation.
6. Add focused schema or contract tests. Run the smallest relevant test set. When no evaluation suite exists for the tool, add registered-tool contract and smoke tests.

Preserve approved names, optional fields, defaults, and boundaries. Propose a contract change for approval before adding stricter rules, renaming fields, or rejecting supported combinations.

## Documentation Contract

| Location | Content |
|---|---|
| Tool description | Use case, side effect, important boundary, and required lookup or skill |
| Field description | Meaning, shape, values, omission/default behavior, lookup source, and conflicts |
| API or SDK comment | Transport semantics or a non-obvious invariant |
| Workflow prompt | Business policy, classification, selection, and workflow-specific decisions |

Use exact JSON names and casing. Define roles such as owner, assignee, editor, and viewer when they differ.

State relevant permission effects. State which authenticated scope supplies the organization or actor.

Use value fields when omission and zero have the same result. Use presence wrappers only when presence changes behavior.

## Field Pattern

Write one compact description in this order:

1. What the field controls.
2. Accepted shape or values.
3. How to resolve IDs.
4. What omission means.
5. Permission or conflict rules.

Example:

> Users or teams that receive task access. Each item contains `id`, `resource` (`user` or `team`), and optional `access` (`viewer` or `editor`). Omitted access becomes `editor`. Resolve user IDs with `search_users`.

## Review Checks

- Schema names, types, enums, required fields, and size limits match the typed input.
- Defaults match the service, not an assumption from `omitempty`.
- The normal authenticated adapter carries user-facing writes.
- The controller and service enforce every documented authorization rule.
- Organization-scoped lookups use authenticated scope.
- Documentation does not infer workflow policy inside a generic tool.
- Tests pin shapes, defaults, roles, and error boundaries without matching whole paragraphs.

## Common Mistakes

- Copying existing behavior into a feature specification when that behavior is the gap.
- Replacing approved field names or defaults with a locally preferred contract.
- Adding pointers to create requests when omission and the zero value are equivalent.
- Saying “private” without naming who can read or edit.
- Promising validation that only one layer performs partially.
- Sampling unrelated tools while missing a close domain pattern.
- Putting workflow-specific classification or ACL selection into generic tool code or SDK documentation.
