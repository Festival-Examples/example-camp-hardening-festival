---
fest_type: task
fest_id: 04_intent_read_migration_and_status_cache.md
fest_name: intent_read_migration_and_status_cache
fest_parent: 08_argv_and_read_paths
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.73165-06:00
fest_updated: 2026-06-15T00:10:27.635672-06:00
fest_tracking: true
---


# Task: Stop intent reads running migration and status all writing the cache (N-8, N-18)

## Objective

Remove `svc.EnsureDirectories` from all four intent read commands so legacy-layout campaigns are never silently migrated by a read, and remove `writeStatusCache` from `runStatusAll` so `camp status all --json` stops creating `.campaign/cache/gitstatus/status.json` on every invocation; clean up the permanently dead `--no-cache` flag.

## Requirements

- [ ] `cmd/camp/intent/list.go:80-82` (verified): `svc.EnsureDirectories(ctx)` call removed from `runIntentList`.
- [ ] `cmd/camp/intent/find.go:71-73` (verified): `svc.EnsureDirectories(ctx)` call removed from `runIntentFind`.
- [ ] `cmd/camp/intent/count.go:50-52` (verified): `svc.EnsureDirectories(ctx)` call removed from `runIntentCount`.
- [ ] `cmd/camp/intent/show.go:69-71` (verified): `svc.EnsureDirectories(ctx)` call removed from `runIntentShow`.
- [ ] Intent reads detect a pending legacy migration and write a warning to stderr pointing to `camp init --repair`; they do not error out (output still flows on stdout).
- [ ] `EnsureDirectories` is retained on write paths: `intent add`, `intent create`, `intent promote`, `intent move`, `intent edit`. Verify each callers' `RunE` still calls it; do not remove it from those paths.
- [ ] `cmd/camp/status_all.go:117` (verified): `writeStatusCache(campRoot, allStatuses)` call removed from `runStatusAll`.
- [ ] `cmd/camp/status_all.go:41-42` (verified): `statusAllNoCache` flag declaration removed.
- [ ] `cmd/camp/status_all.go:50` (verified): `--no-cache` flag registration removed from `init()`.
- [ ] Help text in `statusAllCmd.Long` is updated to remove the `--no-cache` line.
- [ ] `writeStatusCache` itself is deleted from `status_all.go`; the `fsutil` and `time` imports are removed if they become unused.
- [ ] Tests: four named tests documented below.

## Implementation

### Background: intent reads running migration (N-8)

The issue is in all four read commands. Here is `cmd/camp/intent/list.go:79-82` (verified at worktree `bc2ad1f`):

```go
svc := intent.NewIntentService(campaignRoot, resolver.Intents())

// Ensure directories exist and migrate legacy layout
if err := svc.EnsureDirectories(ctx); err != nil {
    return camperrors.Wrap(err, "ensuring intent directories")
}
```

`EnsureDirectories` (verified in `internal/intent/migration.go:50-78`) calls `ensureCanonicalIntentRoot` which calls `migrateLegacyIntentRoot` if it finds `workflow/intents/` state. This rename tree runs unlocked during a read. Two concurrent `camp intent list` calls can race the same rename operations with no lock. The second-pass reviewer reproduced this live: `camp intent list --format json` with a legacy-layout campaign migrated the tree silently.

The correct migration surface is `camp init --repair`. Reads should warn but not mutate.

### Step 1: Add a legacy-layout detection helper

Add a pure-read function to the intent service (or as a package-level helper) that returns whether a legacy migration is pending, without performing it:

```go
// PendingLegacyMigration returns true if the legacy workflow/intents layout exists
// and contains intent state that would be migrated by EnsureDirectories.
// It reads only; it does not create directories or move files.
func (s *IntentService) PendingLegacyMigration() (bool, error) {
    moves, err := s.PlanLegacyIntentRootMigration()
    if err != nil {
        return false, err
    }
    return len(moves) > 0, nil
}
```

`PlanLegacyIntentRootMigration` (verified in `internal/intent/migration.go:130-187`) is already a pure-read planner that returns the moves without executing them. This helper is a one-liner on top of it.

### Step 2: Replace `EnsureDirectories` in the four read commands

In each of the four read commands, replace the `EnsureDirectories` block with a warn-on-pending-migration check:

```go
// Detect legacy layout without migrating it.
if pending, err := svc.PendingLegacyMigration(); err == nil && pending {
    fmt.Fprintf(os.Stderr, "warning: legacy intent layout detected at workflow/intents/; run 'camp init --repair' to migrate\n")
}
```

This is a best-effort warning. A stat error inside `PlanLegacyIntentRootMigration` (e.g. a permission error) returns a non-nil error, and the `err == nil &&` guard simply skips the warning rather than failing the command. Output still flows normally.

The exact same pattern applies to `find.go:71-73`, `count.go:50-52`, and `show.go:69-71`. Each file already has `"fmt"` and `"os"` in scope (or add `"os"` to the import group if missing).

### Step 3: Verify EnsureDirectories is retained on write paths

Search for `EnsureDirectories` callers in the worktree:

```bash
grep -rn "EnsureDirectories" cmd/camp/intent/
```

At the verified worktree the callers are `list.go`, `find.go`, `count.go`, `show.go` (the four reads). Write paths (`add.go`, `create.go`, `promote.go`, `move.go`, `edit.go`) do not currently call it; verify. The design intent is that writes call it so the directory structure exists. If any write path lacks the call, add it. If none do, note that the write paths rely on `os.MkdirAll` inside the actual move/write helpers and no explicit EnsureDirectories is needed.

### Step 4: Remove writeStatusCache from status all

The issue is in `cmd/camp/status_all.go:117` (verified):

```go
// Cache results
writeStatusCache(campRoot, allStatuses)
```

And the dead flag machinery at lines 41-42 and 50:

```go
// Line 41-42 (var block):
statusAllNoCache   bool

// Line 50 (init):
statusAllCmd.Flags().BoolVar(&statusAllNoCache, "no-cache", false, "Skip cache and refresh")
```

`statusAllNoCache` is declared, registered, and never read. There is no cache-read path in the file (verified by searching for `readStatusCache` or any `os.ReadFile` referencing the status cache path: none found).

The decision: **delete the write and the flag**. Rationale: (a) no consumer reads the cache file, (b) writing it on every `--json` read dirtied git status during polling loops, (c) implementing the read path is a separate design decision outside this sequence's scope. Document the decision in a code comment where `writeStatusCache` was called:

```go
// Status cache write removed (camp-hardening R18/N-18): the --no-cache flag was
// dead (no read path existed) and writing during --json reads dirtied git status.
// If a cache is needed in future, implement the read path and honor the flag then.
```

Delete the `writeStatusCache` function body and remove the `statusAllNoCache` var and flag registration. Update `statusAllCmd.Long` to remove the `--no-cache` example line.

Check if `fsutil` and `time` imports become unused after removing `writeStatusCache`; `fsutil.WriteFileAtomically` was called only there. `time` is used by `statusAllCache.Timestamp` in the struct definition; since the struct is also only used in `writeStatusCache`, the struct definition and `time.Time` import can be removed too if nothing else uses them.

### Tests

**Test 1: intent list against a legacy-layout fixture mutates nothing**

```go
func TestIntentList_LegacyLayout_NoMutation(t *testing.T) {
    dir := t.TempDir()
    // Create a fake legacy workflow/intents/inbox/foo.md
    legacyInbox := filepath.Join(dir, "workflow", "intents", "inbox")
    os.MkdirAll(legacyInbox, 0755)
    os.WriteFile(filepath.Join(legacyInbox, "foo.md"), []byte("---\nid: foo\ntitle: Foo\ntype: task\nstatus: inbox\n---\nContent\n"), 0644)

    // Snapshot directory tree
    before := dirTree(t, dir)

    // Create service and call the read path
    svc := intent.NewIntentService(dir, filepath.Join(dir, ".campaign", "intents"))
    pending, _ := svc.PendingLegacyMigration()
    // ... exercise List path ...

    after := dirTree(t, dir)
    if before != after {
        t.Errorf("read mutated the filesystem:\nbefore: %s\nafter: %s", before, after)
    }
    if !pending {
        t.Error("expected PendingLegacyMigration to return true")
    }
}
```

Where `dirTree` walks the directory and returns a sorted snapshot of all paths and sizes.

**Test 2: two concurrent reads race-free by construction**

This test does not need to prove the absence of races formally (that was the lock-free concern with the old EnsureDirectories path); it just verifies neither goroutine returns an error and neither mutates the tree:

```go
func TestIntentList_ConcurrentReads_NoMutation(t *testing.T) {
    dir := setupLegacyFixture(t)
    before := dirTree(t, dir)

    var wg sync.WaitGroup
    for range 5 {
        wg.Add(1)
        go func() {
            defer wg.Done()
            svc := intent.NewIntentService(dir, filepath.Join(dir, ".campaign", "intents"))
            svc.PendingLegacyMigration() // would have been a race with EnsureDirectories
        }()
    }
    wg.Wait()

    after := dirTree(t, dir)
    if before != after {
        t.Error("concurrent reads mutated the filesystem")
    }
}
```

**Test 3: status all --json writes no file**

```go
func TestStatusAll_JSON_NoCache(t *testing.T) {
    dir := t.TempDir()
    cacheFile := filepath.Join(dir, ".campaign", "cache", "gitstatus", "status.json")

    // Exercise the runStatusAll path (or the writeStatusCache deletion path).
    // After: assert the cache file does not exist.
    if _, err := os.Stat(cacheFile); !os.IsNotExist(err) {
        t.Error("status all --json wrote a cache file")
    }
}
```

**Test 4: reads in a read-only campaign dir do not error from attempted writes**

```go
func TestIntentList_ReadOnlyDir_NoError(t *testing.T) {
    dir := t.TempDir()
    // Make the directory read-only after populating a canonical layout.
    os.Chmod(dir, 0555)
    defer os.Chmod(dir, 0755)

    svc := intent.NewIntentService(dir, filepath.Join(dir, ".campaign", "intents"))
    // List should not attempt to create any directories.
    _, err := svc.List(context.Background(), &intent.ListOptions{})
    // We expect either an empty list or a not-found type error, NOT a permission error
    // from attempted directory creation.
    if err != nil && strings.Contains(err.Error(), "permission denied") {
        t.Errorf("read path attempted a write in a read-only dir: %v", err)
    }
}
```

### Edge cases

- **Half-migrated legacy layout (some dirs moved, some not)**: `PlanLegacyIntentRootMigration` returns moves for what remains; the warning fires and points to repair. The user's data is safe in its current partial state.
- **Canonical layout (no legacy)**: `PlanLegacyIntentRootMigration` returns nil moves; the warning is skipped; reads proceed normally.
- **`svc.EnsureDirectories` called from `camp init --repair`**: this path is intentional and must not be changed. Verify `internal/scaffold/repair.go` calls it; leave that call in place.

### Out of scope

- The `jsoncontract` adoption for intent commands (no `--json` flag, only `--format json`) is sequence 09/10.
- Implementing a real cache-read path for `--no-cache` is a future design decision; this task only removes the dead flag and the unconditional write.
- The `EnsureDirectories` write-path behavior itself (the seven-dir scaffold and legacy-root relocation) is not redesigned here; it stays correct for write callers.

## Done When

- [ ] All requirements met
- [ ] `go test -count=1 -short ./cmd/camp/intent/... ./internal/intent/...` passes in both profiles
- [ ] `grep -n "EnsureDirectories" cmd/camp/intent/list.go cmd/camp/intent/find.go cmd/camp/intent/count.go cmd/camp/intent/show.go` returns zero matches
- [ ] `grep -n "writeStatusCache\|statusAllNoCache\|no-cache" cmd/camp/status_all.go` returns zero matches
- [ ] Running `camp intent list` against a legacy-layout fixture prints the warning to stderr and outputs correct data to stdout without modifying any files (`git status --porcelain` shows clean)