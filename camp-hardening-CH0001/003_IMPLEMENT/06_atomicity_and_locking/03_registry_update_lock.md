---
fest_type: task
fest_id: 03_registry_update_lock.md
fest_name: registry_update_lock
fest_parent: 06_atomicity_and_locking
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.69572-06:00
fest_tracking: true
---

# Task: Registry Read-Modify-Write Lock (REG-1)

## Objective

Introduce `config.UpdateRegistry(ctx, mutate func(*Registry) error) error` that holds a file lock across the entire load-mutate-save cycle, then migrate all five mutating callers to it so concurrent camp invocations cannot silently lose registry entries.

## Requirements

- [x] **REG-1a**: `config.UpdateRegistry` introduced in `internal/config/registry.go`; acquires `fsutil.AcquireFileLock(registryPath+".lock")`, re-loads the registry inside the lock, calls the mutate function, saves atomically on success, releases the lock on return.
- [x] **REG-1b**: `cmd/camp/register.go` (lines ~121 and ~164) migrated to `UpdateRegistry`.
- [x] **REG-1c**: `internal/clone/register.go` (lines ~32-43) migrated.
- [x] **REG-1d**: `internal/scaffold/init.go` (lines ~378-382) migrated.
- [x] **REG-1e**: `cmd/camp/switch.go` (lines ~46, ~69, ~98) migrated: the switch command loads twice (once to check active, once to get the target) and saves once; refactor to load once inside `UpdateRegistry`.
- [x] Read-only callers (`LoadRegistry` in list/show paths) are NOT changed; only mutating paths go through the lock.
- [x] Context cancellation inside `UpdateRegistry` is honored: if `ctx.Done()` fires while waiting for the lock, the call returns `ctx.Err()` without modifying the registry.
- [x] Lock-contention timeout behavior is inherited from `fsutil.AcquireFileLock`'s built-in 5-second timeout; callers do not need to set their own deadline.
- [x] Existing tests remain green in both profiles.
- [x] New tests: concurrent-update test (two goroutines registering different campaigns; both present afterward); context cancellation inside the lock returns without mutation.

## Implementation

### Background: why the registry needs a lock

`LoadRegistry` and `SaveRegistry` are currently unguarded. When two camp processes run simultaneously - the norm in agent loops where the festival-app shells `camp register` while another process runs `camp init` - they both read the registry snapshot, mutate their own copy in memory, and write atomically. The atomic write prevents file corruption, but the second writer's save replaces the first writer's registration. The finding (REG-1) cites `cmd/camp/register.go:121,164`, `internal/clone/register.go:32-43`, `internal/scaffold/init.go:384-388`, and `cmd/camp/switch.go:~98`.

The pattern to copy is `links.WithLock` in `internal/workitem/links/save.go:72-97`: acquire `fsutil.AcquireFileLock(path+".lock")`, re-load inside the lock so the mutation sees the most recent disk state, apply the mutation, save atomically, release the lock.

### Step 1: Verify line numbers against the worktree

```bash
grep -n "LoadRegistry\|SaveRegistry" \
  projects/worktrees/camp/camp-hardening/cmd/camp/register.go \
  projects/worktrees/camp/camp-hardening/internal/clone/register.go \
  projects/worktrees/camp/camp-hardening/internal/scaffold/init.go \
  projects/worktrees/camp/camp-hardening/cmd/camp/switch.go
```

Expected hits (may have drifted from review baseline; relocate by symbol if needed):
- `register.go`: `LoadRegistry` at ~121, `SaveRegistry` at ~164
- `clone/register.go`: `LoadRegistry` at ~32, `SaveRegistry` at ~43
- `scaffold/init.go`: `LoadRegistry` at ~378, `SaveRegistry` at ~382 (inside an `if` block with a swallowed error)
- `switch.go`: two `LoadRegistry` calls (~46, ~69) and one `SaveRegistry` (~98)

Note: `internal/project/add.go` was cited in the second-pass findings as a potential fifth caller. Verify:

```bash
grep -n "LoadRegistry\|SaveRegistry" \
  projects/worktrees/camp/camp-hardening/internal/project/add.go
```

If it calls the registry, add it to the migration list.

### Step 2: Read the links.WithLock pattern

Read `internal/workitem/links/save.go:68-97` (verified clean in the second pass; copy this shape, do not modify it):

```go
func WithLock(ctx context.Context, root string, fn func(*Links) error) error {
    if err := ctx.Err(); err != nil {
        return err
    }
    if err := os.MkdirAll(linksDir(root), 0o755); err != nil {
        return camperrors.Wrap(err, "create links dir")
    }
    path := LinksPath(root)
    release, err := fsutil.AcquireFileLock(ctx, path+".lock")
    if err != nil {
        return err
    }
    defer release()

    registry, err := loadLocked(ctx, root)
    if err != nil {
        return err
    }
    if err := fn(registry); err != nil {
        if errors.Is(err, ErrSkipSave) {
            return nil
        }
        return err
    }
    return saveLocked(ctx, root, registry)
}
```

`UpdateRegistry` follows this shape exactly, but operates on the global registry path rather than a campaign-local directory.

### Step 3: Introduce UpdateRegistry in registry.go

Add the following function to `internal/config/registry.go`. The lock file path is `RegistryPath() + ".lock"`:

```go
// UpdateRegistry holds an exclusive lock on the registry for the duration of fn,
// re-loads the registry inside the lock so fn sees the latest on-disk state,
// applies fn's mutations, and saves atomically on success.
// Reads (LoadRegistry) do not need to go through UpdateRegistry; only mutating
// operations should use it.
func UpdateRegistry(ctx context.Context, fn func(*Registry) error) error {
    if err := ctx.Err(); err != nil {
        return err
    }
    lockPath := RegistryPath() + ".lock"
    release, err := fsutil.AcquireFileLock(ctx, lockPath)
    if err != nil {
        return camperrors.Wrap(err, "acquiring registry lock")
    }
    defer release()

    reg, err := LoadRegistry(ctx)
    if err != nil {
        return err
    }
    if err := fn(reg); err != nil {
        return err
    }
    return SaveRegistry(ctx, reg)
}
```

Key design points:
- `LoadRegistry` is called inside the lock so the mutation always sees the latest disk state, even if another process wrote between the caller's pre-lock read and now.
- `SaveRegistry` already uses `fsutil.WriteFileAtomically` (verified at `registry.go:81`).
- The lock file lives alongside the registry: `~/.obey/campaign/registry.json.lock`. This is the same convention `AcquireFileLock` uses in `links` and other subsystems.

### Step 4: Migrate register.go

In `cmd/camp/register.go`, replace the current load-register-save sequence with `UpdateRegistry`. The current flow (approximately `:121-164`):

```go
reg, err := config.LoadRegistry(ctx)
if err != nil {
    return err
}
// ... prompts and validation ...
if err := reg.Register(cfg.ID, name, absPath, ctype); err != nil {
    return camperrors.Wrap(err, "failed to register campaign")
}
if err := config.SaveRegistry(ctx, reg); err != nil {
    return err
}
```

Replace with:

```go
if err := config.UpdateRegistry(ctx, func(reg *config.Registry) error {
    // ... any pre-mutation validation that can run inside the lock ...
    if err := reg.Register(cfg.ID, name, absPath, ctype); err != nil {
        return camperrors.Wrap(err, "failed to register campaign")
    }
    return nil
}); err != nil {
    return err
}
```

Important: any user-facing prompts (the "replace with new path?" confirmation) must happen BEFORE entering `UpdateRegistry`, because `AcquireFileLock` is non-blocking to the user and the lock should not be held while waiting for stdin. Move the prompt and the pre-check reads outside the `UpdateRegistry` call; only the actual mutation goes inside.

Revised pattern for `register.go`:

```go
// Pre-lock: read-only checks and prompts (no mutation)
reg, err := config.LoadRegistry(ctx)
if err != nil {
    return err
}
if existing, exists := reg.GetByName(name); exists && existing.Path != absPath {
    // ... prompt the user ...
    if !confirmed {
        return nil
    }
}
// ... other pre-checks ...

// Locked mutation
if err := config.UpdateRegistry(ctx, func(reg *config.Registry) error {
    // Re-check inside the lock in case another process mutated between our read and now.
    // If the user already confirmed, we can proceed even if the stale entry is gone.
    reg.UnregisterByID(existingIDToRemove) // if applicable
    return reg.Register(cfg.ID, name, absPath, ctype)
}); err != nil {
    return err
}
```

### Step 5: Migrate clone/register.go

The pattern in `internal/clone/register.go:32-43` is a simpler load-register-save with no prompts. Convert directly:

```go
// Before
reg, err := config.LoadRegistry(ctx)
if err != nil {
    return err
}
if err := reg.Register(id, name, path, campType); err != nil {
    return err
}
return config.SaveRegistry(ctx, reg)

// After
return config.UpdateRegistry(ctx, func(reg *config.Registry) error {
    return reg.Register(id, name, path, campType)
})
```

### Step 6: Migrate scaffold/init.go

In `internal/scaffold/init.go` at approximately `:378-382`, the registry save has a swallowed error:

```go
reg, err := config.LoadRegistry(ctx)
if err == nil {
    if err := reg.Register(...); err == nil {
        _ = config.SaveRegistry(ctx, reg)
    }
}
```

After migration, surface the error properly:

```go
if err := config.UpdateRegistry(ctx, func(reg *config.Registry) error {
    return reg.Register(id, name, path, campType)
}); err != nil {
    // Registration failure during init is a warning, not a fatal error.
    // The campaign was initialized; only the registry entry is missing.
    fmt.Fprintf(os.Stderr, "camp: warning: failed to register campaign: %v\n", err)
}
```

The existing behavior swallows errors silently (`_ = config.SaveRegistry`). The migration makes the error visible as a warning but does not fail init. This is the correct semantic since the campaign directory is already created.

### Step 7: Migrate switch.go

In `cmd/camp/switch.go`, there are two `LoadRegistry` calls (one to find the active campaign, one to find the target) and one `SaveRegistry` to update `LastAccess`. The entire flow needs to go through a single `UpdateRegistry` call. Review the full function before migrating to ensure the pre-lock reads are separated from the mutation:

```bash
# Read the full switch command RunE to understand the flow
cat projects/worktrees/camp/camp-hardening/cmd/camp/switch.go
```

The migration shape: do the two LoadRegistry reads outside the lock for the user-facing lookup, then wrap only the `UpdateLastAccess` + `SaveRegistry` in `UpdateRegistry`:

```go
if err := config.UpdateRegistry(ctx, func(reg *config.Registry) error {
    reg.UpdateLastAccess(targetID)
    return nil
}); err != nil {
    // Log the error but do not fail the switch; last-access update is best-effort.
    fmt.Fprintf(os.Stderr, "camp: warning: failed to update last access: %v\n", err)
}
```

### Step 8: Run the both-profile gate

```bash
just build
BUILD_TAGS=dev just build
go test -count=1 -short \
  ./internal/config/... \
  ./internal/clone/... \
  ./internal/scaffold/... \
  ./cmd/camp/...
go test -tags dev -count=1 -short \
  ./internal/config/... \
  ./internal/clone/... \
  ./internal/scaffold/... \
  ./cmd/camp/...
```

### Step 9: Add concurrency tests

Test placement: host `t.TempDir()` unit tests.

**Test: two concurrent goroutines registering different campaigns - both present afterward**

In `internal/config/registry_test.go` (add to existing test file):

```go
func TestUpdateRegistryConcurrent(t *testing.T) {
    // Point the registry to a temp dir for this test.
    dir := t.TempDir()
    t.Setenv("CAMP_REGISTRY_PATH", filepath.Join(dir, "registry.json"))

    ctx := context.Background()
    var wg sync.WaitGroup
    wg.Add(2)
    errs := make([]error, 2)

    go func() {
        defer wg.Done()
        errs[0] = UpdateRegistry(ctx, func(reg *Registry) error {
            return reg.Register("id-a", "campaign-a", filepath.Join(dir, "a"), CampaignTypeProduct)
        })
    }()
    go func() {
        defer wg.Done()
        errs[1] = UpdateRegistry(ctx, func(reg *Registry) error {
            return reg.Register("id-b", "campaign-b", filepath.Join(dir, "b"), CampaignTypeProduct)
        })
    }()
    wg.Wait()

    for i, err := range errs {
        if err != nil {
            t.Errorf("goroutine %d: %v", i, err)
        }
    }

    reg, err := LoadRegistry(ctx)
    if err != nil {
        t.Fatalf("LoadRegistry: %v", err)
    }
    if _, ok := reg.GetByID("id-a"); !ok {
        t.Error("id-a missing after concurrent registration")
    }
    if _, ok := reg.GetByID("id-b"); !ok {
        t.Error("id-b missing after concurrent registration")
    }
}
```

**Test: context cancellation during lock acquisition returns error without mutation**

```go
func TestUpdateRegistryContextCancellation(t *testing.T) {
    dir := t.TempDir()
    t.Setenv("CAMP_REGISTRY_PATH", filepath.Join(dir, "registry.json"))

    ctx, cancel := context.WithCancel(context.Background())
    cancel() // cancel immediately

    err := UpdateRegistry(ctx, func(reg *Registry) error {
        t.Error("mutate fn should not be called on cancelled context")
        return nil
    })
    if err == nil {
        t.Error("expected error on cancelled context")
    }
}
```

## Edge Cases

- **Pre-lock reads in register.go**: the user-facing prompts must stay outside the lock. Do not hold the lock while waiting for stdin; that would cause the 5-second timeout to fire for any interactive command.
- **`RegistryPath()`**: this function reads `CAMP_REGISTRY_PATH` env var and applies XDG logic. The lock path must be `RegistryPath() + ".lock"` computed at call time, not at import time, so test overrides via `t.Setenv` work correctly.
- **scaffold/init.go error handling**: the existing code swallows the save error. The migration makes it a visible warning. This is a behavior change, but the correct one; do not silently swallow errors from `UpdateRegistry`.
- **Verified negatives**: `internal/config/registry.go` itself (`LoadRegistry`, `SaveRegistry`) are NOT changed. `links.WithLock` and `QuarantineBroken` are certified clean and must not be touched.

## Out of Scope

- Do not refactor the `Registry` struct schema.
- Do not lock the nav-index rebuild; that is sequence 10 task 06 (NAV-3).
- Do not change `LoadRegistry` callers that are read-only (list, show, get, detect, status-all lookups).
- `cmd/camp/switch.go` also contains read-only uses of `LoadRegistry` for lookups; only the `UpdateLastAccess` mutation path needs the lock.

## Completion Notes

- Project commit: `2759cc6` (`fix: lock registry update transactions`).
- Added `config.UpdateRegistry(ctx, mutate)` around an exclusive `registry.json.lock` lock with load-mutate-save inside the critical section.
- Migrated the required callers: `cmd/camp/register.go`, `internal/clone/register.go`, `internal/scaffold/init.go`, `cmd/camp/switch.go`, and the applicable `cmd/camp/project/add.go` last-access update path.
- Expanded the migration to every production direct registry mutation found by search: `cmd/camp/registry/commands.go`, `cmd/camp/unregister.go`, `cmd/camp/list.go`, `cmd/camp/intent/add.go`, and `cmd/camp/attach/resolver.go`.
- Kept prompts and campaign selection outside the lock; `cmd/camp/register.go` re-checks name/path conflicts inside the locked snapshot and asks the user to retry if a new unconfirmed conflict appears.
- Added `TestUpdateRegistryConcurrent` and `TestUpdateRegistryContextCancellationWhileWaitingForLock` in `internal/config/registry_test.go`.

## Verification

- `rg -n "config\\.SaveRegistry|saveRegistry" cmd internal --glob '!**/*_test.go' || true`
- `grep -n "SaveRegistry" cmd/camp/register.go internal/clone/register.go internal/scaffold/init.go cmd/camp/switch.go cmd/camp/project/add.go || true`
- `go test -count=1 -short ./internal/config/... ./internal/clone/... ./internal/scaffold/... ./cmd/camp/...`
- `go test -tags dev -count=1 -short ./internal/config/... ./internal/clone/... ./internal/scaffold/... ./cmd/camp/...`
- `go test -v -count=1 -run 'TestUpdateRegistry' ./internal/config`
- `just build stable`
- `just build dev`
- `git diff --check`

## Done When

- [x] All requirements met.
- [x] `UpdateRegistry` function exists in `internal/config/registry.go`.
- [x] All five mutating callers migrated (`register.go`, `clone/register.go`, `scaffold/init.go`, `switch.go`, and `project/add.go` if applicable).
- [x] Both-profile gate passes.
- [x] Concurrent-update test: two goroutines, both campaigns present afterward.
- [x] Context-cancellation test passes.
- [x] `grep -n "SaveRegistry" cmd/camp/register.go internal/clone/register.go internal/scaffold/init.go cmd/camp/switch.go` returns zero results (all replaced by `UpdateRegistry`).
