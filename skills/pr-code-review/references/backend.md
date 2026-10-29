# Backend Review Checks

Use for diffs under `backend/`.

## Metal Backend

- Check both `black-sabbath-api` gateway routes and `metallica-api` handlers when contracts change.
- Preserve org and tenant scoping on every data path.
- Use context-aware calls for external services and database operations.
- Keep MongoDB models, indexes, and JSON/BSON tags aligned.
- Treat event-driven cleanup as at-least-once and retry-prone.

## Gotchas

| Area | Check |
| --- | --- |
| API contracts | Gateway route, Go handler, frontend caller, tests |
| MongoDB | Index coverage, tenant filter, projection, sort, pagination |
| Auth | Role checks, org ownership, cross-tenant access tests |
| Events | Idempotency, retry safety, poison-message behavior |
| Redis | Key scope, TTL, invalidation, stale reads |
| External APIs | Timeout, cancellation, retry, partial failure |
| Go errors | Wrapped errors, status codes, nil and zero-value behavior |
| Background work | Cleanup order, durable state, duplicate processing |

## Tests

Expect focused tests for changed behavior. Useful targets include:

- Controller tests for auth, validation, and status codes.
- Service tests for edge cases and partial failures.
- Event tests for duplicate delivery and retry behavior.
- Query tests for tenant scoping and missing data.
