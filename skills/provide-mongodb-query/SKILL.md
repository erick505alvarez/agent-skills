---
name: provide-mongodb-query
description: Provide MongoDB shell queries for the user to run themselves, instead of executing them via tools. Use when the user asks for a Mongo query, asks for help querying a collection, or explicitly invokes this skill. Adds a safety/friction layer by surfacing the query rather than executing it.
disable-model-invocation: true
---

# Provide MongoDB Query

The user prefers to run MongoDB queries themselves. Your job is to **produce** the query, not execute it.

## Workflow

1. Clarify the intent if ambiguous: target collection, filter, projection, sort, limit.
2. Determine whether the collection is organization-scoped. If it is, require the organization ID and include `org: ObjectId("...")` in every preview, read, aggregation, and write filter.
3. Identify the indexes available on the target collection (check the model/schema definitions in the repo if needed). Pick a filter+sort shape that uses an existing index with `org` as its leading field for organization-scoped collections.
4. If no suitable index exists, adjust the query to use one or stop and recommend an index. Do not provide an inefficient query by default.
5. Output the query in a single copy-pastable shell code block, terminated with `.toArray()` (or the appropriate cursor-materializing method — see below).

## Output Format

Always emit a `mongosh`-compatible code block. Examples:

```javascript
db.deals.find(
  { org: ObjectId("..."), status: "active" },
  { name: 1, status: 1, updatedAt: 1 }
).sort({ updatedAt: -1 }).limit(50).toArray()
```

```javascript
db.companies.aggregate([
  { $match: { org: ObjectId("...") } },
  { $sort: { createdAt: -1 } },
  { $limit: 100 },
  { $project: { name: 1, createdAt: 1 } }
]).toArray()
```

### Terminator rules

- `find(...)` → append `.toArray()`
- `aggregate([...])` → append `.toArray()`
- `findOne(...)`, `countDocuments(...)`, `distinct(...)`, `updateOne/Many(...)`, `deleteOne/Many(...)` → leave as-is (they already return a value, not a cursor)

## Index Coverage (mandatory)

By default, every query you provide must be backed by an index. The only exception is a query the user explicitly requests after receiving the collection-scan warning below.

- Filter fields and sort fields must align with a real compound index in the order the index defines.
- Prefer projections that limit returned fields, especially for wide documents.
- Always include a reasonable `.limit(...)` for `find` queries unless the user asks for all results.
- For aggregations, put `$match` first and ensure it hits an index; put `$sort` early enough to use an index when possible.

If you cannot confirm an index exists:

> ⚠️ No suitable index for this filter/sort. I won't provide a query that collection-scans `<collection>`. Recommended index: `{ field1: 1, field2: -1 }`.

Do not provide the inefficient query unless the user explicitly asks for it after seeing the warning.

## Organization Scoping (mandatory)

Treat collections containing an `org` field as organization-scoped.

- Include an equality predicate such as `org: ObjectId("...")` in every query, including `findOne`, `countDocuments`, `distinct`, aggregations, previews, and writes.
- Use an existing compound index whose leading field is `org`. An `_id` predicate alone does not replace organization scoping.
- If the organization ID is missing, ask for it and do not emit the query.
- Never broaden a query by removing `org` to make another index usable.
- If the user explicitly requests a cross-organization query, state that it bypasses normal tenant scoping, warn about its expected scan/index cost, and prefer a per-organization query. Provide the cross-organization form only after that explicit request.

## ObjectId and Date Literals

Use shell-native constructors, not strings:

- IDs: `ObjectId("...")`
- Dates: `ISODate("2025-01-01T00:00:00Z")` or `new Date("...")`

## Writes

For any write (`update*`, `delete*`, `bulkWrite`, `replaceOne`):

1. First provide a matching `find(...).toArray()` or `countDocuments(...)` so the user can preview the affected documents.
2. Then provide the write query in a separate code block.
3. Note that the user should review the preview before running the write.

## Do Not

- Do not execute the query yourself via MCP, shell, or any tool. Provide it as text.
- Do not wrap multiple queries in one code block — one query per block so each is independently copy-pastable.
- Do not omit `.toArray()` on cursor-returning queries.
- Do not provide a query against an organization-scoped collection without an `org` equality filter unless the user explicitly requested cross-organization scope.
- Do not silently provide a collection scan or an index-incompatible filter/sort as a fallback.
