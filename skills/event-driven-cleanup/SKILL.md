---
name: event-driven-cleanup
description: Patterns for safe event-driven resource cleanup in distributed systems. Use when implementing cascading deletes, bulk cleanup, or any operation where MongoDB mutations depend on successful event publishing for downstream cleanup (e.g., Vespa, S3).
---

# Event-Driven Cleanup

## Core Principle

**Publish events before mutating the database.** If event publishing fails, return an error so the client can retry without leaving orphaned data in downstream systems (Vespa, S3, etc.).

## Operation Order

```
1. Validate  — confirm access, existence, permissions (read-only)
2. Collect   — gather IDs of resources to clean up (read-only)
3. Publish   — enqueue cleanup events; fail the request if publishing fails
4. Delete    — remove from MongoDB only after events are confirmed published
```

If MongoDB deletion fails in step 4, the system is temporarily inconsistent but downstream cleanup still proceeds via the published events. This is the safer failure mode.

## Bulk Cleanup via Fan-Out

When deleting many resources (e.g., all sessions in a project), avoid publishing N individual events from the controller. Instead:

1. **Controller** publishes a single bulk event (e.g., `EventBulkSessionDelete`) containing all resource IDs.
2. **Worker handler** receives the bulk event and fans out individual events using `PublishMessagesV2` for batch publishing.
3. **Individual handlers** process each resource independently with SQS retry semantics.

This gives retry semantics at both levels — the bulk event retries the entire fan-out, and each individual event retries its own cleanup.

```
Controller                    Worker (bulk handler)         Worker (per-item handler)
    │                              │                              │
    ├─ Publish BulkDelete ────────►│                              │
    │  (single event)              ├─ Fan out via                 │
    │                              │  PublishMessagesV2 ─────────►│
    │                              │                              ├─ Clean up Vespa
    ├─ DeleteMany (MongoDB) ◄──────┤                              ├─ Clean up S3
    │  (only after publish)        │                              │
```

## Example: Project Deletion with Session Cleanup

```go
// 1. Validate
project, err := svc.GetProject(ctx, orgID, projectID)
// check permissions...

// 2. Collect
sessions := findSessionsByProject(ctx, orgID, projectID)
sessionIDs := extractIDs(sessions)

// 3. Publish — fail request if this fails
bulkEvt := events.EventBulkSessionDelete{SessionIDs: sessionIDs, OrgID: orgID}
if err := publish(ctx, bulkEvt); err != nil {
    return http.StatusInternalServerError // nothing deleted, safe to retry
}

// 4. Delete — events are published, now mutate MongoDB
sessionsCollection.DeleteMany(ctx, bson.M{"_id": bson.M{"$in": sessionIDs}})
projectsCollection.DeleteOne(ctx, bson.M{"_id": projectID})
```

## When to Apply

- Cascading deletes (parent resource owns child resources indexed in Vespa/S3)
- Bulk resource removal tied to downstream cleanup
- Any controller that publishes events AND deletes from MongoDB in the same request
