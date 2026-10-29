---
name: write-script
description: Create Go scripts in backend/scripts/ following project conventions. Use when the user asks to write, create, or scaffold a new backend script, migration, backfill, CLI tool, or one-off data operation.
---

# Write Script

## Location & Module Setup

All scripts live under `backend/scripts/<script-name>/`.

### 1. Create the directory and module

```bash
mkdir -p backend/scripts/<script-name>
cd backend/scripts/<script-name>
go mod init <script-name>
```

### 2. Add the `lib/go` dependency with a local replace

```go
// go.mod
module <script-name>

go 1.26.1

require (
    github.com/getmetal/metal-backend/lib/go v0.0.0-20251211002212-bc66d9e01772
    github.com/joho/godotenv v1.5.1
    go.mongodb.org/mongo-driver v1.17.3
)

replace github.com/getmetal/metal-backend/lib/go => ../../lib/go
```

### 3. Register in the workspace

Add the script to `backend/go.work`:

```
use (
    ./scripts/<script-name>
    // ... existing entries
)
```

### 4. Tidy with private module flags

```bash
GOPRIVATE=github.com/getmetal/* GONOSUMCHECK=github.com/getmetal/* go mod tidy
```

### 5. Create `.env.example`

Document required environment variables. Never commit `.env` (already in `.gitignore`).

```bash
# Only needed for --env staging
STAGING_DATABASE_URL=

# Only needed for --env prod
PROD_DATABASE_URL=
```

## main.go Conventions

### File-level doc comment

Every script starts with a package-level doc comment explaining:
- What the script does (one paragraph)
- Idempotency guarantees
- Prerequisites (env vars, .env setup)
- Usage examples with all flag combinations
- Interactive behavior (if any)

### Standard CLI flags

Use Go's `flag` package. Common flags across scripts:

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `--env` | string | `"dev"` | Target environment: dev, staging, prod |
| `--org` | string | `""` | Scope to a single org ObjectID |
| `--exclude` | string | `""` | Comma-separated org IDs to skip |
| `--dryRun` | bool | `false` | Log what would happen without writes |
| `--batchSize` | int | `500` | MongoDB cursor batch size |
| `--flushDelay` | int | `100` | Milliseconds between bulk write flushes |

### Standard helper functions

These are duplicated per script (not shared via lib):

```go
var validEnvs = map[string]bool{"dev": true, "staging": true, "prod": true}

func getMongoURI(env string) string {
    switch env {
    case "dev":
        return "mongodb://localhost:27017"
    case "staging":
        return os.Getenv("STAGING_DATABASE_URL")
    case "prod":
        return os.Getenv("PROD_DATABASE_URL")
    default:
        return ""
    }
}

func connectMongo(ctx context.Context, uri string) *mongo.Client {
    client, err := mongo.Connect(ctx, options.Client().ApplyURI(uri))
    if err != nil {
        log.Fatalf("Failed to connect to MongoDB: %v", err)
    }
    if err := client.Ping(ctx, nil); err != nil {
        log.Fatalf("Failed to ping MongoDB: %v", err)
    }
    log.Println("Connected to MongoDB")
    return client
}

func getDBName(env string) string {
    switch env {
    case "prod", "staging":
        return "metal-control"
    case "dev":
        return "blacksabbath"
    default:
        return ""
    }
}

func dryRunPrefix(dryRun bool) string {
    if dryRun {
        return "[DRY RUN] "
    }
    return ""
}
```

### Graceful shutdown

Handle SIGINT/SIGTERM to disconnect MongoDB cleanly:

```go
ctx, cancel := context.WithCancel(context.Background())
client := connectMongo(ctx, mongoURI)

sigs := make(chan os.Signal, 1)
signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

go func() {
    sig := <-sigs
    log.Printf("Received signal: %s, shutting down...", sig)
    cancel()
    shutdownCtx, shutdownCancel := context.WithTimeout(
        context.Background(), 30*time.Second,
    )
    defer shutdownCancel()
    if err := client.Disconnect(shutdownCtx); err != nil {
        log.Printf("Error disconnecting: %v", err)
    }
}()
```

### Org discovery

Use `Distinct` on the target collection to find orgs, then iterate per-org:

```go
func distinctOrgIDs(ctx context.Context, db *mongo.Database,
    singleOrg *primitive.ObjectID,
    exclude map[primitive.ObjectID]struct{},
) ([]primitive.ObjectID, error) {
    // If --org is set, return just that one
    // Otherwise Distinct("org", bson.M{}) on the target collection
    // Filter out excluded orgs
}
```

## MongoDB Best Practices

### Index-aware queries

Verify queries are backed by existing indexes. Check the collection's index definitions before writing filters. Prefer compound index key order in query filters.

### Cursor options for long-running scripts

```go
opts := options.Find().
    SetProjection(projection).
    SetBatchSize(int32(batchSize)).
    SetNoCursorTimeout(true) // prevents 10-min server timeout
```

### Lightweight projections

Only project the fields needed for the operation — define a minimal struct:

```go
type dealDoc struct {
    ID     primitive.ObjectID `bson:"_id"`
    Org    primitive.ObjectID `bson:"org"`
    Name   string             `bson:"name"`
    Status types.DealStatus   `bson:"status"`
}
```

### Bulk writes with periodic flushing

Accumulate `WriteModel` operations and flush in batches to bound memory and control write throughput:

```go
const bulkFlushThreshold = 1000

func flushBulkWrite(ctx context.Context, coll *mongo.Collection,
    models *[]mongo.WriteModel, label string,
) error {
    if len(*models) == 0 {
        return nil
    }
    res, err := coll.BulkWrite(ctx, *models, options.BulkWrite().SetOrdered(false))
    if err != nil {
        return fmt.Errorf("bulk write failed for %s: %w", label, err)
    }
    log.Printf("  %s bulk write: matched=%d, modified=%d",
        label, res.MatchedCount, res.ModifiedCount)
    *models = (*models)[:0]
    return nil
}
```

Add a configurable `--flushDelay` sleep between flushes to avoid saturating MongoDB IOPS in production.

### Idempotent update filters

When backfilling a missing field, always include the missing-field predicate in the update filter — not just `_id`. Interactive prompts and batched flushes widen the gap between read and write, so a concurrent update can set the field before the BulkWrite executes.

```go
// BAD — can overwrite a concurrently-set value
SetFilter(bson.M{"_id": deal.ID})

// GOOD — no-ops if the field was set between Find and BulkWrite
SetFilter(bson.M{"_id": deal.ID, "statusConfig": nil})
```

With `SetOrdered(false)`, a no-op match simply yields `matchedCount=0` for that document.

### Stdin-closed sentinel for interactive scripts

When a script reads from stdin for interactive prompts, a closed stdin (e.g., piped input runs out, accidental EOF) must propagate as a fatal error — not a silent nil return. Define a sentinel error and return it from the prompt path:

```go
var errStdinClosed = fmt.Errorf("stdin closed unexpectedly")

// In the prompt function:
if !scanner.Scan() {
    return result, errStdinClosed
}
```

In the org loop, catch it with `errors.Is` to break immediately and exit non-zero after the summary. A partial backfill that exits 0 is indistinguishable from a complete one.

## Output & Logging

### Startup banner

Log the resolved configuration so the operator can verify before proceeding:

```go
log.Println("=== script-name ===")
log.Printf("Mode:      %s", mode)
log.Printf("Env:       %s", env)
log.Printf("Database:  %s", dbName)
log.Println("===================")
```

### Per-org progress

```go
log.Printf("org %d/%d (%s)", i+1, len(orgs), org.Hex())
```

### Summary at end

Log totals for scanned, modified, skipped, errored counts and elapsed time.

### ANSI colors for interactive prompts

Use color codes to distinguish operation types visually:

```go
const (
    colorReset   = "\033[0m"
    colorCyan    = "\033[36m"
    colorMagenta = "\033[35m"
    colorYellow  = "\033[33m"
    colorBold    = "\033[1m"
    colorDim     = "\033[2m"
)
```

Colors work in both `fmt.Printf` (stdout) and `log.Printf` (stderr) in VS Code's terminal.

## Result tracking

Define a result struct with an `add` method for per-org accumulation:

```go
type backfillResult struct {
    Scanned      int
    AutoMatched  int
    UserSelected int
    Skipped      int
    Errors       int
}

func (r backfillResult) add(other backfillResult) backfillResult { ... }
```

## Checklist

- [ ] `go.mod` has `replace` directive pointing to `../../lib/go`
- [ ] Script is registered in `backend/go.work`
- [ ] `.env.example` documents required vars
- [ ] `--dryRun` flag skips all writes
- [ ] Queries leverage existing MongoDB indexes
- [ ] Cursors use `SetNoCursorTimeout(true)`
- [ ] Bulk writes flush periodically (not unbounded)
- [ ] Update filters include the missing-field predicate (not just `_id`)
- [ ] Interactive scripts return a sentinel error on stdin close (not nil)
- [ ] Graceful shutdown handles SIGINT/SIGTERM
- [ ] Startup banner logs resolved config
- [ ] Summary printed at end with counts and elapsed time
- [ ] `go build ./...` and `go vet ./...` pass
- [ ] No compiled binaries committed to git
