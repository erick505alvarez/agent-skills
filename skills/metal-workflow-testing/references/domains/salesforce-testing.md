# Salesforce Testing Patterns

Load this only when the workflow targets Salesforce. Extends the universal patterns in `../test-patterns.md`.

## Scenario Templates

### Prepare-Review Payload Shape

| Scenario | Check | Expected |
|----------|-------|----------|
| Field-level review items | `salesforce_prepare_review_payload` output | Each item has correct editability metadata |
| Reference fields read-only | `reference` type fields | Not editable |
| Multipicklist read-only | `multipicklist` type fields | Not editable |
| Picklist editable | `picklist` type fields with describe-reported options | Editable with matching option values |

### Duplicate Detection

| Scenario | Check | Expected |
|----------|-------|----------|
| Duplicate warning surfaced | `salesforce_validate_operations` with a create that matches an existing record | Returns `potential_duplicate` warning |
| Workflow holds duplicates | Workflow receives duplicate warning | Create held back for review, not written |
| Duplicate preflight unavailable | `duplicate_preflight_unavailable` warning | Workflow acknowledges no duplicate signal |

### Composite Graph Behavior

| Scenario | Check | Expected |
|----------|-------|----------|
| Dry-run write | `salesforce_entity_write` with `dryRun: true` | Returns Composite request structure without calling Salesforce |
| Transactional mode | `transactional` write mode | Single Composite Graph request |
| Best-effort mode | `best_effort` write mode | Standard Composite request |

### File Attachment

| Scenario | Check | Expected |
|----------|-------|----------|
| Data entity source | `salesforce_upload_document` with `data_entity_id` | File attached to record's Files related list |
| Artifact source | `salesforce_upload_document` with `artifact_id` | File attached from sandbox artifact |
| Title dedup | Re-run with same title | Returns `alreadyExisted: true`, no duplicate upload |
| New version | Re-run with different title | New file uploaded, `alreadyExisted: false` |

### Account Resolution

| Scenario | Check | Expected |
|----------|-------|----------|
| Fuzzy match | Company name with punctuation/casing variation | Resolves to correct account |
| Alias mapping | Known firm name variant | Resolves via alias |
| No match | Completely unknown company | Graceful handling, not a crash |

## Persona Evaluation Additions

- Salesforce-specific: verify that object labels and field labels are used in reviewer-facing display, not API names.
- Salesforce-specific: verify that picklist values are displayed as labels, not API values.

## Common Failure Modes

- Duplicate creates written without validation check.
- Account resolution fails on exact-match when fuzzy match would succeed.
- File attachment fails because `parent_record_id` is wrong or file-incapable.
- Prepare-review generates whole-operation envelopes instead of field-level items.
