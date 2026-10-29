# Gotcha Searches

Use these searches during review. Adapt names to the diff.

## Required Searches

| Change type | Search for |
| --- | --- |
| New or changed function | All callers and tests for the function name |
| New API route | Gateway route, frontend request, auth middleware, tests |
| Changed request or response type | Type definitions, serializers, consumers, fixtures |
| Database query | Indexes, tenant filters, pagination, sort order |
| Event emission | Consumers, retry behavior, idempotency keys |
| Cache change | All cache keys, invalidation paths, stale readers |
| Feature flag | Default state, rollout paths, hidden dependencies |
| Permission change | Role checks, org scoping, unauthorized tests |
| Delete or cleanup flow | Cascades, background jobs, retries, partial failure |
| UI state change | Loading, empty, error, disabled, and race states |

## Commands

```bash
rg "SymbolName" frontend backend
rg "route-or-endpoint" frontend backend
rg "featureFlagName|permissionName|eventName" frontend backend
rg "TODO|FIXME|HACK" changed/path nearby/path
```

## Review Questions

- Does the implementation satisfy each Linear acceptance criterion?
- Can a caller pass nil, empty, stale, unauthorized, or cross-tenant data?
- Does a retry, refresh, double-click, or concurrent request break behavior?
- Does the change require cache invalidation, event cleanup, or index changes?
- Does the test fail on the likely bug, not only cover the happy path?
