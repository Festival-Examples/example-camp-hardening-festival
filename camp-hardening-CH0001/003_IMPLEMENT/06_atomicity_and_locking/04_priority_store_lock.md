---
fest_type: task
fest_id: 04_priority_store_lock.md
fest_name: priority_store_lock
fest_parent: 06_atomicity_and_locking
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.695918-06:00
fest_tracking: true
---

# Task: Priority Store Read-Modify-Write Lock (R12 workitem priority)

## Objective

Introduce `priority.WithLock(ctx, storePath, fn)` mirroring `links.WithLock` so that concurrent `camp workitem priority` invocations and TUI priority mutations cannot silently lose each other's writes.

## Requirements

- [x] **R12-prio-a**: `priority.WithLock(ctx context.Context, storePath string, fn func(*Store) error) error` introduced in `internal/workitem/priority/store.go`; acquires `fsutil.AcquireFileLock(storePath+".lock")`, re-loads the store inside the lock, calls fn, saves (or deletes) on success.
- [x] **R12-prio-b**: CLI command in `internal/commands/workitem/priority.go` (currently `:85-98`) migrated to `WithLock`; the `priority.Load` + `priority.Set/Clear` + `priority.SaveOrDelete` sequence replaced.
- [x] **R12-prio-c**: TUI `assignPriority` in `internal/workitem/tui/update.go` (~`:289-308`) migrated to `WithLock`.
- [x] **R12-prio-d**: TUI `clearPriority` in `internal/workitem/tui/update.go` (~`:311-329`) migrated to `WithLock`.
- [x] `priority.Prune` is NOT moved here. The Prune call in the TUI at `update.go:38-39` (during item load) stays as-is; Prune moves to explicit write paths in sequence 08 task 03. `WithLock` must be the only door for `Set` and `Clear` mutations so sequence 08's change slots in cleanly.
- [x] `SaveOrDelete` contract preserved: `WithLock` internally calls `SaveOrDelete` so an empty store deletes the file rather than writing an empty JSON object.
- [x] Existing `priority` package tests (`priority_test.go`, `store_test.go`, `integration_test.go`) remain green in both profiles.
- [x] New tests: concurrent Set and Clear from two goroutines lose no entries; existing `priority_test.go` tests remain green.

## Implementation

### Background: why the priority store needs a lock

Festival-app polls `camp workitem ... --json` continuously and exposes priority management directly. The TUI also mutates priority on keypress. The current pattern in both paths is `priority.Load(storePath)` then `priority.SaveOrDelete(storePath, store)` with no file lock between them. Two concurrent saves (say, app calls the CLI while the TUI is open) load the same snapshot and the second save silently overwrites the first. The second pass confirmed this at `priority.go:85-98` and `tui/update.go:296-297,318-319`.

The pattern to copy is `links.WithLock` at `internal/workitem/links/save.go:72-97` (certified clean in the second pass). The priority store is the same "mutable contract file" class as the links store; it just missed the locking treatment.

### Step 1: Verify line numbers against the worktree

```bash
grep -n "priority.Load\|priority.SaveOrDelete\|priority.Set\|priority.Clear" \
  projects/worktrees/camp/camp-hardening/internal/commands/workitem/priority.go \
  projects/worktrees/camp/camp-hardening/internal/workitem/tui/update.go
```

Expected results at `bc2ad1f`:

| File | Lines | Pattern |
|------|-------|---------|
| `commands/workitem/priority.go` | ~85-98 | `Load` then `Set/Clear` then `SaveOrDelete` |
| `tui/update.go` | ~296-297 | `Set` then `SaveOrDelete` in `assignPriority` |
| `tui/update.go` | ~318-319 | `Clear` then `SaveOrDelete` in `clearPriority` |

Also verify the Prune call in `tui/update.go` (do NOT migrate it; just confirm its location):

```bash
grep -n "priority.Prune\|priority.Apply" \
  projects/worktrees/camp/camp-hardening/internal/workitem/tui/update.go
```

### Step 2: Read the links.WithLock pattern

Read `internal/workitem/links/save.go:72-97` (do not modify it; it is certified clean). The key elements:

1. Context check at the top.
2. `os.MkdirAll` for the directory (ensures the lock file can be created).
3. `fsutil.AcquireFileLock(ctx, path+".lock")`.
4. `defer release()`.
5. Load inside the lock.
6. Call fn with the loaded state.
7. Save atomically on success.

### Step 3: Introduce priority.WithLock

Add to `internal/workitem/priority/store.go`:

```go
// WithLock holds an exclusive lock on the priority store for the duration of fn.
// It re-loads the store inside the lock so fn always sees the latest on-disk state.
// On success, the store is saved atomically (or the file is deleted if the store is empty).
// fn may return any error; a non-nil error cancels the save.
func WithLock(ctx context.Context, storePath string, fn func(*Store) error) error {
    if err := ctx.Err(); err != nil {
        return err
    }
    if err := os.MkdirAll(filepath.Dir(storePath), 0o755); err != nil {
        return camperrors.Wrap(err, "create priority store directory")
    }
    release, err := fsutil.AcquireFileLock(ctx, storePath+".lock")
    if err != nil {
        return err
    }
    defer release()

    store, err := Load(storePath)
    if err != nil {
        return err
    }
    if err := fn(store); err != nil {
        return err
    }
    return SaveOrDelete(storePath, store)
}
```

Required imports to add to `store.go` (if not already present):

```go
"context"
"os"
"path/filepath"

"github.com/Obedience-Corp/camp/internal/fsutil"
```

Note: `camperrors` is already imported. `Load`, `SaveOrDelete`, `filepath`, and `os` are already used in `store.go`. Add only what is missing.

### Step 4: Migrate the CLI command

In `internal/commands/workitem/priority.go`, the current RunE mutation path:

```go
storePath := priority.StorePath(root)
store, err := priority.Load(storePath)
if err != nil {
    return camperrors.Wrap(err, "loading priority store")
}
if clear {
    priority.Clear(store, wi.Key)
} else {
    priority.Set(store, wi.Key, level)
}
if err := priority.SaveOrDelete(storePath, store); err != nil {
    return camperrors.Wrap(err, "saving priority store")
}
```

Replace with:

```go
storePath := priority.StorePath(root)
if err := priority.WithLock(ctx, storePath, func(store *priority.Store) error {
    if clear {
        priority.Clear(store, wi.Key)
    } else {
        priority.Set(store, wi.Key, level)
    }
    return nil
}); err != nil {
    return camperrors.Wrap(err, "updating priority store")
}
```

The `ctx` used here is the cobra command context (`cmd.Context()`). Verify that the command's RunE has access to `ctx`; if the current code uses `context.Background()`, switch to `cmd.Context()`. Check:

```bash
grep -n "ctx\|context\." \
  projects/worktrees/camp/camp-hardening/internal/commands/workitem/priority.go | head -20
```

### Step 5: Migrate the TUI assignPriority

In `internal/workitem/tui/update.go`, `assignPriority`:

```go
// Current (approximately :296-297):
priority.Set(m.priorityStore, item.Key, p)
if err := priority.SaveOrDelete(m.priorityStorePath(), m.priorityStore); err != nil {
    m.exitPriorityMode()
    cmd := m.setStatus("save failed: "+err.Error(), true)
    return m, cmd
}
m.allItems = priority.Apply(m.priorityStore, m.allItems)
```

After migration the in-memory `m.priorityStore` is used for the `Apply` call, but the mutation goes through `WithLock`. The challenge is that `WithLock` re-loads from disk, so `m.priorityStore` may be stale after the lock. Update `m.priorityStore` from the locked load:

```go
var updated *priority.Store
if err := priority.WithLock(context.Background(), m.priorityStorePath(), func(store *priority.Store) error {
    priority.Set(store, item.Key, p)
    updated = store
    return nil
}); err != nil {
    m.exitPriorityMode()
    cmd := m.setStatus("save failed: "+err.Error(), true)
    return m, cmd
}
m.priorityStore = updated
m.allItems = priority.Apply(m.priorityStore, m.allItems)
```

Note: `context.Background()` is used here because the TUI model does not carry a context. This is pre-existing behavior in the TUI (no context propagation). The lock has a built-in 5-second timeout; in practice priority saves are sub-millisecond, so the background context is safe.

### Step 6: Migrate the TUI clearPriority

Apply the same pattern to `clearPriority` at ~`:311-329`:

```go
var updated *priority.Store
if err := priority.WithLock(context.Background(), m.priorityStorePath(), func(store *priority.Store) error {
    priority.Clear(store, item.Key)
    updated = store
    return nil
}); err != nil {
    m.exitPriorityMode()
    cmd := m.setStatus("save failed: "+err.Error(), true)
    return m, cmd
}
m.priorityStore = updated
m.allItems = priority.Apply(m.priorityStore, m.allItems)
```

### Step 7: Confirm Prune is NOT touched

The Prune call in `tui/update.go` at approximately `:38-39`:

```go
if priority.Prune(m.priorityStore, validKeys) {
    _ = priority.SaveOrDelete(m.priorityStorePath(), m.priorityStore)
}
```

Leave this as-is. Sequence 08 task 03 moves `Prune` out of read paths entirely. The existing `SaveOrDelete` here is unlocked; that is acceptable because the Prune mutation is being removed in sequence 08. Do not add a lock here now; doing so would require sequence 08 to also update the locking logic for a call site that is being deleted.

### Step 8: Run the both-profile gate

```bash
just build
BUILD_TAGS=dev just build
go test -count=1 -short \
  ./internal/workitem/priority/... \
  ./internal/commands/workitem/... \
  ./internal/workitem/tui/...
go test -tags dev -count=1 -short \
  ./internal/workitem/priority/... \
  ./internal/commands/workitem/... \
  ./internal/workitem/tui/...
```

### Step 9: Add concurrency tests

Test placement: host `t.TempDir()` unit tests.

**Test: two goroutines set different keys; neither is lost**

In `internal/workitem/priority/store_test.go` (add to the existing suite):

```go
func TestWithLockConcurrentSet(t *testing.T) {
    dir := t.TempDir()
    storePath := filepath.Join(dir, "workitems.json")
    ctx := context.Background()

    var wg sync.WaitGroup
    wg.Add(2)
    errs := make([]error, 2)

    go func() {
        defer wg.Done()
        errs[0] = WithLock(ctx, storePath, func(s *Store) error {
            Set(s, "key-a", High)
            return nil
        })
    }()
    go func() {
        defer wg.Done()
        errs[1] = WithLock(ctx, storePath, func(s *Store) error {
            Set(s, "key-b", Medium)
            return nil
        })
    }()
    wg.Wait()

    for i, err := range errs {
        if err != nil {
            t.Errorf("goroutine %d: %v", i, err)
        }
    }

    store, err := Load(storePath)
    if err != nil {
        t.Fatalf("Load: %v", err)
    }
    if store.ManualPriorities["key-a"].Priority != High {
        t.Errorf("key-a: expected high, got %v", store.ManualPriorities["key-a"])
    }
    if store.ManualPriorities["key-b"].Priority != Medium {
        t.Errorf("key-b: expected medium, got %v", store.ManualPriorities["key-b"])
    }
}
```

**Test: concurrent Set and Clear on the same key - one wins, final state is consistent**

```go
func TestWithLockConcurrentSetClear(t *testing.T) {
    dir := t.TempDir()
    storePath := filepath.Join(dir, "workitems.json")
    ctx := context.Background()

    const iterations = 50
    var wg sync.WaitGroup
    for i := 0; i < iterations; i++ {
        wg.Add(2)
        go func() {
            defer wg.Done()
            _ = WithLock(ctx, storePath, func(s *Store) error {
                Set(s, "race-key", High)
                return nil
            })
        }()
        go func() {
            defer wg.Done()
            _ = WithLock(ctx, storePath, func(s *Store) error {
                Clear(s, "race-key")
                return nil
            })
        }()
    }
    wg.Wait()

    // The store must be loadable and internally consistent (no partial JSON).
    store, err := Load(storePath)
    if err != nil {
        t.Fatalf("Load after concurrent set/clear: %v", err)
    }
    // Either the key is present with High or absent; either is valid.
    entry := store.ManualPriorities["race-key"]
    if entry.Priority != "" && entry.Priority != High {
        t.Errorf("unexpected priority %q; expected empty or High", entry.Priority)
    }
}
```

## Edge Cases

- **TUI in-memory store**: after `assignPriority` and `clearPriority` migrate to `WithLock`, the local `m.priorityStore` pointer is updated from the locked snapshot (`updated = store`). This means the in-memory store always reflects what was written to disk, even if another process mutated the store between TUI refreshes. This is correct behavior.
- **`SaveOrDelete` inside `WithLock`**: when the fn clears the last entry, `SaveOrDelete` calls `os.Remove(storePath)`. The lock file (`storePath+".lock"`) is a separate file and is not removed by `SaveOrDelete`. The lock file is removed by `defer release()` after `SaveOrDelete` returns. This is the correct order.
- **Empty store first run**: `Load` returns a `NewStore()` when the file does not exist. `WithLock` will call `os.MkdirAll` before acquiring the lock, so the settings directory exists before the lock file is created.
- **Lock file cleanup on crash**: if the process dies while holding the lock, the `.lock` file remains. The stale-lock removal in `fsutil.AcquireFileLock` handles this (30-second threshold). Task 05 in this sequence fixes the TOCTOU race in that removal; task 04 is safe to ship before task 05 because the window is narrow and the failure mode is a 5-second wait, not data loss.

## Out of Scope

- Moving `priority.Prune` out of the TUI load path: that is sequence 08 task 03.
- Moving `priority.Prune` out of `workitem list` reads: that is also sequence 08 task 03.
- The `integration_test.go` vacuous tests noted in the second pass (the filter-safety test at `:63-85` is tautological): those are sequence 12 test hygiene; do not fix here.
- Adding context propagation to the TUI model: pre-existing limitation; not in scope.

## Completion Notes

- Project commit: `5197dfe` (`fix: lock priority store updates`).
- Added `priority.WithLock(ctx, storePath, fn)` in `internal/workitem/priority/store.go`, using `fsutil.AcquireFileLock(storePath+".lock")` around load-mutate-`SaveOrDelete`.
- Migrated `internal/commands/workitem/priority.go`, `assignPriority`, and `clearPriority` to locked transactions.
- Updated the TUI in-memory priority store from the locked snapshot before applying/sorting items.
- Left the existing prune-on-refresh `SaveOrDelete` path unchanged because sequence 08 task 03 removes that read-path mutation.
- Added `TestWithLockConcurrentSet` and `TestWithLockConcurrentSetClear` in `internal/workitem/priority/store_test.go`.

## Verification

- `grep -n "priority.Load\\|priority.SaveOrDelete\\|priority.Set\\|priority.Clear" internal/commands/workitem/priority.go internal/workitem/tui/update.go`
- `grep -n "priority.Prune\\|priority.Apply" internal/workitem/tui/update.go`
- `grep -n "priority.SaveOrDelete" internal/commands/workitem/priority.go internal/workitem/tui/update.go || true`
- `go test -count=1 -short ./internal/workitem/priority/... ./internal/commands/workitem/... ./internal/workitem/tui/...`
- `go test -tags dev -count=1 -short ./internal/workitem/priority/... ./internal/commands/workitem/... ./internal/workitem/tui/...`
- `just build stable`
- `just build dev`
- `git diff --check`

## Done When

- [x] All requirements met.
- [x] `priority.WithLock` function exists in `internal/workitem/priority/store.go`.
- [x] CLI command and both TUI mutation functions migrated.
- [x] `grep -n "priority.SaveOrDelete" internal/commands/workitem/priority.go internal/workitem/tui/update.go` returns zero results for the `assignPriority` and `clearPriority` sites (the Prune site in `update.go` is expected to remain until sequence 08).
- [x] Both-profile gate passes.
- [x] Concurrent Set test: two goroutines, both keys present afterward.
- [x] Concurrent Set/Clear test: store is loadable after 50 iterations.
- [x] Existing `priority_test.go` and `store_test.go` tests remain green.
