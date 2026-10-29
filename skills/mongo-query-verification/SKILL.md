---
name: mongo-query-verification
description: Use when writing or running MongoDB/mongosh queries against nested documents, dynamic field paths, arrays, missing fields, or organization-scoped collections. Also use after any MongoDB query whose counts or projections look suspiciously zero, identical, or otherwise too clean.
---

# Mongo Query Verification

Inspect the actual document shape before writing a projection or aggregation. Verify every dynamic field path and count against a direct, explicit query before trusting the result.

## Verification Recipe

Run this sequence when a query result looks wrong.

```javascript
// 1. One real document, unrestricted fields.
db.<collection>.findOne({ org: ObjectId("ORG_ID") })
```

```javascript
// 2. The exact projected field, restricted to the target path.
const path = "columns." + ObjectId("COLUMN_ID").toHexString();
const projection = { _id: 1 };
projection[path] = 1;
db.<collection>.find({ org: ObjectId("ORG_ID"), listId: ObjectId("LIST_ID") }, projection)
  .limit(5).toArray()
```

```javascript
// 3. A direct existence/missing count using the real path.
db.<collection>.countDocuments({
  org: ObjectId("ORG_ID"),
  listId: ObjectId("LIST_ID"),
  [path]: { $exists: false }
})
```

## Dynamic Field Paths in JavaScript

Build dynamic paths with bracket syntax and `.toHexString()`, then read the nested field through the same key.

```javascript
const columnKey = ObjectId("6a70cad7487d3d0e4bafc4cc").toHexString();
const cellPath = "columns." + columnKey;
const projection = { _id: 1 };
projection[cellPath] = 1;

const rows = db.resourcelistentries.find(
  { org: ObjectId("..."), listId: ObjectId("...") },
  projection
).toArray();

const missing = rows.filter(
  (row) => row.columns?.[columnKey]?.sortKey == null
).length;
```

Do not project a constant and read that constant:

```javascript
// WRONG: `cell` is a constant, not the nested document.
const projection = { _id: 1, cell: 1 };
const rows = db.collection.find(filter, projection).toArray();
const missing = rows.filter((row) => !row.cell).length;
```

## Quick Reference

| Symptom | Check first |
|---|---|
| All counts are zero | Is the field path correct? Did the projection include the real path? |
| All counts are identical | Is the projection selecting a constant or a single field? |
| All rows report missing a field | Is the field nested under `columns.<id>`? |
| `.toHexString()` throws on a column | Is `_id`, `org`, or `listId` present in every projected document? |
| One document has data but the aggregate says otherwise | Compare the aggregate against a direct `find` projection. |

## Guard Optional Identifiers

Check for missing `_id`, `org`, `listId`, or embedded fields before calling `.toHexString()`, `.map()`, or indexing into them. A field that is optional in one document can be absent in another.

For organization-scoped collections, include `org` in every query. Use the `provide-mongodb-query` skill for scoping, index selection, and cursor terminators.
