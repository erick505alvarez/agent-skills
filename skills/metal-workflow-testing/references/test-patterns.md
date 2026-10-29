# Universal Test Patterns

Scenario templates that apply to any Metal workflow, regardless of provider. Generate scenarios from these patterns based on what the workflow definition contains.

## Input Coverage

For each field in `inputSchema`:

| Scenario | Input | Expected |
|----------|-------|----------|
| Required field with valid value | Resolved resource or valid literal | Workflow proceeds past input validation |
| Optional field present | Valid value for the field type | Workflow uses the provided value |
| Optional field absent | Omit the field | Workflow uses default or proceeds without it |
| Required field with empty string | `""` | Input validation rejects |

## HITL Paths

For each `humanInLoop` step:

| Scenario | Action | Expected |
|----------|--------|----------|
| Approve all items | Approve all review items | Run continues to next step |
| Reject | Reject the review | Run becomes `REJECTED` |
| Field edit | Approve with `fieldEdits` | Edited values appear in `approvedPayload.operations[*].value.review.editedValue` |
| Partial approval | Select a subset of review items | Only approved items appear in `approvedPayload.operations` |

## Branch Paths

For each `branch` step:

| Scenario | Input | Expected |
|----------|-------|----------|
| Condition true | Inputs that make the condition evaluate true | `results.branch` is `"then"`, then-arm children execute |
| Condition false | Inputs that make the condition evaluate false | `results.branch` is `"else"`, else-arm children execute |

## Iterator Edges

For each `iterator` step:

| Scenario | Source Data | Expected |
|----------|------------|----------|
| Zero items | Source resolves to empty array | Iterator completes with 0 items, downstream steps handle empty |
| One item | Source resolves to one item | Iterator processes the single item |
| Multiple items | Source resolves to multiple items | All items processed, no items silently dropped |
| Item failure | One item causes child step failure | Iterator records the failure, remaining items still process |

## Template Robustness

| Scenario | Input | Expected |
|----------|-------|----------|
| Missing optional nested field | Input resource lacks a nested field the template references | Template renders without `<no value>` or the workflow handles it gracefully |
| Empty string in text field | `""` for a text field the template references | Template renders empty string, not `<no value>` |
| Null-like resource field | Resource field is null or absent | Template renders without error |

## Output Validation

| Scenario | Check | Expected |
|----------|-------|----------|
| Output sections render | Every `outputConfig` section | Section has non-empty content |
| HTML validation | HTML output sections | No `outputs.sections[].error` |
| Citation presence | Output schema declares citations | Citations exist in run evidence |
| Document generation | Workflow has generateDocument or artifact steps | `outputs.documentFiles` or `outputs.outputFiles` non-empty |
