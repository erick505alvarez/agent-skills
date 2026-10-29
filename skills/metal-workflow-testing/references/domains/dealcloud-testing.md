# DealCloud Testing Patterns

Load this only when the workflow targets DealCloud. Extends the universal patterns in `../test-patterns.md`.

## Scenario Templates

### Prepare-Review Payload Shape

| Scenario | Check | Expected |
|----------|-------|----------|
| Field-level review items | `dealcloud_prepare_review_payload` output | Each item has `operationGroupId`, `fieldKey`, and correct `editable` metadata |
| Choice field editability | Single-select choice fields | Editable as picklists with numeric choice IDs |
| Multi-select read-only | Multi-select fields | Not editable regardless of type |
| Cascading fields read-only | Fields sharing a choice set (e.g., Deal Status / Deal Stage) | Not editable |

### Validation Flow

| Scenario | Check | Expected |
|----------|-------|----------|
| Valid operations | `dealcloud_validate_operations` with valid operations | Returns `valid: true` |
| Invalid operations | `dealcloud_validate_operations` with invalid operations | Returns `valid: false` with field-level errors |
| Workflow handles invalid | Workflow receives invalid validation result | Operations held back, reviewer informed |

### Dependency Tiers

| Scenario | Check | Expected |
|----------|-------|----------|
| Multi-entity writes | Operations with parent-child relationships | Parent operations ordered before child operations in `dealcloud_entity_write_v_2` output |

### Dry-Run Write Preflight

| Scenario | Check | Expected |
|----------|-------|----------|
| Dry-run write | `dealcloud_entity_write_v_2` with `dryRun: true` | Returns dependency-tiered Rows API payloads in `requests` without calling DealCloud |

## Persona Evaluation Additions

- DealCloud-specific: verify that entry type names and field labels are used in reviewer-facing display, not raw IDs.
- DealCloud-specific: verify that choice field values are displayed as labels, not numeric choice IDs.

## Common Failure Modes

- Prepare-review generates whole-operation envelopes instead of field-level items (check for `value.review` metadata).
- Choice fields rendered as raw numeric IDs in reviewer display.
- Dependency ordering wrong: child entity written before parent.
- Validation skipped: operations go straight to review without `dealcloud_validate_operations`.
