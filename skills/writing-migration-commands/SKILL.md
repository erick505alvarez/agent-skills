---
name: writing-migration-commands
description: Use when creating or changing a Go MongoDB migration command under backend/scripts/migration, especially when the command changes org-scoped data, supports dry-run, or processes all organizations.
---

# Writing Migration Commands

## Overview

Migration commands make production data changes. Match the local command structure. Prove safety through focused tests.

## Required Discovery

Read `backend/AGENTS.md`, the shared `internal/runner` package, and at least two comparable commands. Confirm collection BSON models before every MongoDB query.

Use the closest command for flags, options, result counters, logs, and tests. Do not invent command behavior or report a test as passed without its output.

## Command Contract

| Concern | Required pattern |
| --- | --- |
| Flags | Every migration command that queries MongoDB registers `-org`, `-all-orgs`, and `-dry-run`. Use shared runner flags where their meaning fits. Use an explicit `-dry-run=true` default for high-impact migrations. |
| Scope | Require exactly one of `-org` or `-all-orgs`. Validate `-org` before Mongo access. |
| Tenant safety | Include `org` in every Mongo query and every write filter. The only exception is a global collection, such as `orgs` while resolving org IDs. Document the exception in code and test it. |
| Reads | Project required fields, check `cursor.Err()`, and close cursors. |
| Writes | Use narrow `$set`/`$unset` updates. Include observed state in compare-and-set filters. Count zero matches as conflicts. |
| Batching | Bound entry and `$in` lookup batches. Do not retain an unbounded list in memory. |
| Dry run | Run the normal read, validation, and classification path. Skip every write. |
| Results | Return errors from `Run`; the command wrapper calls `log.Fatal`. Stop at the first organization query or processing error. |
| Logs | Emit each organization summary and one `scope=all` summary. Include `dryRun=%t` in all summary logs. |

## Tests

Write tests beside the command. Use Mongo `mtest` to inspect filters and writes.

- Test that `-org`, `-all-orgs`, and `-dry-run` are registered, plus their defaults and invalid combinations.
- Test `-org` and `-all-orgs` selection. In all-org mode, process each org independently.
- Test every Mongo read and write contains `org`, except a documented global-collection lookup.
- Test dry-run and apply behavior. Capture logs and assert `dryRun=true` and `dryRun=false` in per-org and aggregate summaries.
- Test database errors stop processing and unsafe counters return errors.
- Test conflicts, malformed rows, and bounded batches when the command writes many documents.

## Evidence

Use this response shape for validation:

```text
Inspected: <files read, or none>
Executed: <exact command and final output, or none>
Not executed: <checks not run>
```

Report only commands executed in this session and their observed final output. Do not say a command, migration, README, or test exists unless you inspected it. Do not claim `go test` or `git diff --check` passed without execution output.

## Common Mistakes

| Mistake | Required correction |
| --- | --- |
| A startup banner shows mode | Add `dryRun=%t` to every result summary and test both modes. |
| An entry ID or list ID identifies a tenant | Add `org` to the filter. IDs do not replace tenant scope. |
| A command handles one org, so it skips `-all-orgs` | Register `-org`, `-all-orgs`, and `-dry-run` on every command that queries MongoDB. |
| A global collection lookup skips `org` without explanation | Document the collection as global in code and test that lookup. |
| An all-org run needs no extra tests | Test every organization and stop behavior after an error. |
| `SetBatchSize` bounds memory | Flush application batches and chunk lookup IDs. |
| A zero-match update succeeds | Count it as a conflict. |
| A test is likely to pass | Run it and report the command output. |
| The task names a command | Inspect its registration and files. Do not infer that it exists or is correct. |
| A concise response needs validation | Write `Executed: none`. Never substitute a plausible command or result. |

## Red Flags

- “The command likely exists.”
- “Tests passed” without the exact executed command and observed final output.
- “This command only handles one org, so it does not need `-all-orgs`.”
- “This collection is probably global, so it does not need `org`.”
- A test command for an unrelated package.

Stop and use the evidence response shape.

## Review Checklist

1. Confirm every Mongo operation includes `org`, or is a documented global-collection lookup.
2. Confirm the command registers `-org`, `-all-orgs`, and `-dry-run`.
3. Confirm dry-run reaches classification but no writes.
4. Confirm writes use compare-and-set filters and count conflicts.
5. Confirm logs state `scope` and `dryRun` for per-org and aggregate results.
6. Run focused tests and `git diff --check`.
