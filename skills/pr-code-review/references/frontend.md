# Frontend Review Checks

Use for diffs under `frontend/`.

## Metal Frontend

- Preserve Next.js App Router server/client boundaries.
- Use SWR for client-side server data.
- Include `credentials: "include"` for authenticated client requests.
- Keep request types aligned with backend response types.
- Preserve PropelAuth cookie behavior for server-side requests.
- Follow shadcn/ui, Radix, Tailwind, and local component patterns.

## Gotchas

| Area | Check |
| --- | --- |
| SWR | Stable keys, correct mutation, write invalidation |
| Forms | Disabled state, validation errors, double-submit behavior |
| Tables | Pagination, sorting, filtering, row selection, empty state |
| Server components | No client-only hooks or browser APIs in server code |
| Client components | Required `"use client"` and no secret server data exposure |
| Layout | Mobile fit, text overflow, loading and error states |
| Auth | Unauthorized and expired session behavior |
| Analytics | No duplicate noisy events on render loops |

## Tests

Expect tests for behavior changes when practical. Useful targets include:

- Hook behavior around loading, error, and mutation states.
- Component behavior for empty, disabled, and permission-denied states.
- Request helpers when API contracts or auth handling change.
