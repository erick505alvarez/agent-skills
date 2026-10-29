---
name: writing-comments
description: Write succinct, comprehensive code comments in colloquial language. Use when adding, editing, or reviewing comments in any code file (any language). Triggers when writing inline comments, docblocks, JSDoc, GoDoc, Python docstrings, or explanatory notes inside source code.
---

# Writing Comments

Write code comments that are **short, plain-spoken, and useful**. A reader should understand the *why* in one glance without wading through prose.

## Core Rules

1. **Explain why, not what.** The code already shows what it does. Comments cover intent, trade-offs, constraints, and gotchas.
2. **Cap it at 2-3 lines.** Aim for one line, two if it earns it, three at the absolute most. Reaching for a fourth line is a signal, not a green light: you're being exhaustive instead of succinct. Cut to the reason that actually matters, or split the rest into trailing inline notes (see Anchor Comments) instead of stacking more sentences in the block. Cut filler words.
3. **Be comprehensive within the cap.** Succinct doesn't mean vague — say the real reason in those 2-3 lines, don't hand-wave it. It does mean you don't get to capture every nuance; drop the least essential one before you drop the cap.
4. **Use colloquial language.** Write like you'd explain it to a teammate at a whiteboard. No corporate filler, no academic tone.
5. **Skip the obvious.** If the code reads naturally, no comment — unless it's a signpost comment (see below).
6. **Present tense, state-based.** Describe what is true, not what changed or what used to be.
7. **Name concrete things, not abstractions.** Every noun in the comment should point at a function, variable, constant, or operation the reader can grep for. No pronouns standing in for ideas, no jargon shorthand, no labels you invented while writing.
8. **Anchor comments to specific lines.** Prefer a short anchor line above a construct plus trailing inline notes (`code // why`) over multi-line block comments above a region. Reserve block comments for things that apply to the whole construct — and even those stay inside the 2-3 line cap.

## What "Colloquial" Means Here

- Contractions are fine (`don't`, `can't`, `we're`).
- Plain words over fancy ones: "skip" not "circumvent", "fix" not "remediate", "weird" not "anomalous".
- Talk to the reader directly when it helps: `// we hit this when the user lands without a session`.
- Avoid hedging fluff: drop "basically", "essentially", "in order to", "for the purpose of".

## Anti-Patterns

Avoid these — they waste tokens and add noise:

```ts
// Increment the counter
counter++;

// Loop through users
for (const user of users) { ... }

// Return the result
return result;

// This function fetches the user
function getUser(id: string) { ... }
```

Also avoid:
- **Change-history narration**: `// previously this used X, now it uses Y`. Just describe what it does.
- **Restating the signature**: `// takes a string and returns a number`. The types say that.
- **Vague hand-waves**: `// handle edge case`. Which one? Why?
- **Transient references.** Anything that can rot without notice:
  - Ticket IDs alone: `// see ENG-7441` — tickets get closed, renamed, archived. Describe the actual problem.
  - Line numbers or file paths in other files: `// see utils.ts:142`, `// matches logic on line 88`. Lines shift, files move.
  - "See the function above/below", "the block on the next page", "as we discussed in the PR".
  - Author/date stamps: `// added by Erick 2024-03-12`. Git blame already knows.

  If the context genuinely lives elsewhere, **name the symbol, not the location**: `// mirrors normalizeRow() in deals/utils.ts` survives a refactor; `// see line 412` doesn't.

## Make Every Noun Concrete

A comment can be succinct, present-tense, and explain *why* — and still leave the reader guessing — when its nouns are abstractions instead of actual code symbols. The reader has to reconstruct what each label refers to from surrounding context. If a chain of causation is non-trivial, every link in the chain must name a real thing in scope.

**Bad** — abstract nouns force the reader to decode:

```go
// First page only — the token carries the decision into resume pages so
// a write between requests can't flip the verdict mid-paginate.
```

What's "the decision"? What's a "resume page"? Whose "verdict"? Three labels the writer invented; three lookups the reader has to perform against surrounding code.

**Good** — every noun points at something the reader can grep for:

```go
// Run the count on page 1 only. If we counted again on later pages,
// inserts between requests could push the result over the threshold and
// disable the custom sort mid-paginate. The cursor was built against the
// custom sort order, so falling back to _id ordering would dupe or skip rows.
```

"Count", "threshold", "custom sort", "cursor", "_id ordering" are all concrete things named elsewhere in the same function.

### Clarity check

Before keeping a comment, scan it for:

- **Pronouns / abstract nouns referring to ideas.** `the decision`, `the state`, `the verdict`, `it`, `this`. Replace with the variable, constant, or operation by name.
- **Jargon shorthand.** `mid-paginate`, `carries forward`, `resume page`, `re-check`. Cute, but opaque to anyone walking in cold.
- **Consequence without cause.** `would dupe or skip rows` — trace the mechanism: *what changes*, *what assumption breaks*, *why those combine to produce the wrong output*.
- **Labels the writer invented.** If the word doesn't appear in the code, picking it up by reading is a guess.

If you can't replace every abstract noun with a concrete one, the comment is leaning on context the reader may not have. Split it, anchor each clause to its actual symbol, or move it closer to the line it describes.

## Anchor Comments to Specific Lines

When a comment applies to one line of a construct, attach it to that line. A short anchor immediately above + trailing inline notes on the lines that need explanation beats a multi-line block comment above a region — the reader stops having to mentally map sentence-to-line.

**Bad** — block above; reader maps each sentence onto specific lines:

```go
// _activeOrder_<col> = sortOrder when active, null when archived.
// Carrying -1 forward would work for the sort (the _isArchived field
// already groups them) but null keeps the field meaning "real sort
// position or nothing" instead of leaking the sentinel into the cursor.
bson.E{Key: sort.ActiveOrderFieldName(), Value: bson.M{
    "$cond": bson.A{
        bson.M{"$eq": bson.A{sortOrderRef, -1}},
        nil,
        sortOrderRef,
    },
}},
```

**Good** — short anchor + trailing inline on the value that earns the why:

```go
// _activeOrder_<col> = sortOrder when active, null when archived
bson.E{Key: sort.ActiveOrderFieldName(), Value: bson.M{
    "$cond": bson.A{
        bson.M{"$eq": bson.A{sortOrderRef, -1}},
        nil, // null when sortOrder == -1
        sortOrderRef,
    },
}},
```

Inline notes are also where the **"what we configure vs what the library/runtime does"** distinction belongs. Put it on the operator or value that triggers the implicit behavior:

```go
"$eq": bson.A{sortOrderRef, -1}, // Mongo BSON sort: false<true, so $sort ASC puts archived rows last
```

Reserve block comments for things that apply to the whole construct — explaining a pipeline stage's purpose, an exported function's contract, a non-obvious algorithm. Anything narrower belongs on the line it describes.

## Signpost Comments (Keep These)

A **signpost comment** is a single line above a logical step in a longer function. It labels what the next block does so a reader can scan the function without parsing every line.

These are not "why" comments and they don't need a gotcha. Their job is navigation: they name the step, branch, or fallback the code block implements.

**Keep signpost comments when simplifying or editing.** Do not delete them just because the code below is readable in isolation. In a 50-line loop or switch, that readability does not carry — the signpost is what makes the structure visible.

```go
// Check for descending prefix
descending := false
if strings.HasPrefix(field, "-") { ... }

// Validate column exists and get the column definition
col, exists := columnMap[field]
if !exists {
    // Try to find by ID if field is an ObjectID
    if colId, err := primitive.ObjectIDFromHex(field); err == nil { ... }
}
```

**Signpost vs noise:**

| Keep (signpost) | Cut (noise) |
|-----------------|-------------|
| Labels a distinct step in a multi-step function | Sits on a one-liner with no surrounding structure |
| Names a non-obvious branch or fallback path | Restates what a variable assignment obviously does |
| Helps scan a long `for`/`switch`/nested `if` block | Duplicates an exported function's doc comment |

Signposts stay short — one line, no block. If a step needs a *why*, add a separate why comment or inline note; don't turn the signpost into a paragraph.

### When the Reasoning Runs Long

A real justification can have three or four true parts. Writing all of them is still a violation if the block runs past 3 lines — the cap forces you to find the one or two parts the reader actually needs and cut the rest, even when every part is individually correct.

**Bad** — every reason is true, but the block keeps growing to fit them all:

```go
// Tag each field (ak:/u:/s:) so two fields with the same raw value (e.g. an
// apiKeyID equal to a userID) can't collide into the same buffer bytes and
// share a bucket. This matters most for the shared org-wide burst bucket,
// where every client type funnels through one key namespace and an
// untagged collision could leak one client's rate limit into another's.
// Per-client dedicated buckets already partition by a unique prefix, so
// here the tags are just a harmless extra safety net.
```

**Good** — keep the one reason that would actually confuse a reader (why the tags exist at all); drop the secondary case where they're merely redundant:

```go
// Tag each field (ak:/u:/s:) so two fields sharing a raw value can't
// collide into the same bucket key — real risk in the shared org-wide
// burst bucket, where every client type funnels through one namespace.
if apiKeyID != nil {
```

## Good Examples

```ts
// Vespa rejects payloads >10MB, so chunk before upload.
const chunks = chunkPayload(doc, MAX_VESPA_BYTES);

// Race: PropelAuth sometimes returns 401 right after login while the
// token propagates. Retry once before bailing.
const user = await fetchWithRetry(() => getUser(), { attempts: 2 });

// We mutate a copy here — the parent passes this array as a prop.
const sorted = [...rows].sort(byCreatedAt);

// Stripe webhooks can fire twice for the same event; idempotency key
// keeps the DB write from doubling up.
await db.upsert({ key: event.id, ... });
```

```go
// MongoDB index is on (orgID, createdAt); query in that order or it scans.
filter := bson.M{"org_id": orgID, "created_at": bson.M{"$gte": since}}

// Caller already holds the lock — don't re-acquire or we deadlock.
func (s *Store) writeLocked(...) { ... }
```

```python
# Bedrock throttles aggressively above 10 req/s — back off exponentially.
time.sleep(2 ** attempt)
```

## Decision Flow

Before writing a comment, ask:

1. **Is the code self-explanatory?** → No comment.
2. **Is there a non-obvious reason, constraint, or gotcha?** → Comment the *why* in one line.
3. **Would a teammate be confused or break this in 6 months?** → Add the warning.

## Special Cases

- **TODO / FIXME / HACK**: include enough context to act on it. `// TODO: switch to streaming once SDK >=4.2 lands` beats `// TODO: fix later`.
- **Public APIs (exported funcs, components, types)**: a one-line summary is fine — describe purpose and any non-obvious contract (e.g. "caller owns the cleanup", "returns nil when empty").
- **Magic numbers / regexes**: always comment the source or reasoning. `// matches Stripe customer IDs (cus_...)`
- **Workarounds**: describe the *symptom* and the trigger condition so the comment stays useful even if the tracking link rots. Pin to a stable artifact (upstream library version, public GitHub issue URL) rather than an internal ticket ID. `// workaround: AG Grid v31 fires onCellValueChanged twice for paste — drop duplicates by rowId+colId` beats `// workaround for ENG-7441`.

## When Editing Existing Code

- **Don't delete relevant comments.** If a comment looks stale, verify before removing.
- **Protect signpost comments.** When tightening or simplifying comments, keep existing single-line signposts that label steps in multi-step functions. Only remove one if the labeled block was deleted or merged into an adjacent step.
- **Don't narrate your edit in a comment.** The diff already shows the change.
- **Tighten verbose block comments** opportunistically when touching nearby code, but don't go on a comment-rewriting spree unrelated to the task. The simplification target is multi-line blocks that restate nuance — not one-line signposts that aid scanning.
