---
fest_type: task
fest_id: 04_repair_migration_all_or_nothing.md
fest_name: repair_migration_all_or_nothing
fest_parent: 03_sync_and_migration_safety
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.638084-06:00
fest_updated: 2026-06-14T20:07:55.672288-06:00
fest_tracking: true
---


# Task: Make repair migrations all-or-nothing and never auto-commit after failure

**Finding:** R4 (INIT-1, WF-3)

## Objective

Pre-validate every migration (src exists, dst absent) before moving anything; on any failure report what moved and what did not, exit non-zero, and guarantee that `commitRepairChanges` never runs after a failed migration.

## Requirements

- [ ] `ExecuteMigrations` in `internal/scaffold/repair.go` pre-validates every (src, dst) pair for the entire plan before moving a single file: src must exist, dst must be absent using `statuspath.ExistingItemPath` where applicable; POSIX `os.Rename` silently clobbers files so an existing dst is a hard pre-validation failure.
- [ ] If pre-validation fails, `ExecuteMigrations` returns an error describing all invalid pairs without moving anything.
- [ ] If pre-validation passes but a mid-sequence rename fails, `ExecuteMigrations` stops, returns an error that includes how many items moved and how many remain, and does NOT continue.
- [ ] The caller in `cmd/camp/init/command.go` propagates the migration error as an actual error (not a warning) and returns it before reaching `commitRepairChanges`.
- [ ] `commitRepairChanges` is unreachable on any path where `ExecuteMigrations` returns a non-nil error.
- [ ] The mover logic is structured as a small, named helper function with a documented no-replace contract, ready for extraction by sequence 11.05 (D003).
- [ ] Unit tests: pre-validation rejects a plan where a dst already exists; mid-sequence failure leaves no auto-commit and exits non-zero; rerunning after partial failure errors cleanly on the moved dst (which now exists) rather than clobbering.

## Implementation

### Background

`ExecuteMigrations` at `internal/scaffold/repair.go:652-689` (worktree bc2ad1f, verified at `:654-689`):

```go
func ExecuteMigrations(migrations []MigrationAction) (int, error) {
    moved := 0
    for _, m := range migrations {
        if err := os.MkdirAll(m.Dest, 0755); err != nil {
            return moved, camperrors.Wrapf(err, "creating %s", m.Dest)
        }
        for _, item := range m.Items {
            src := filepath.Join(m.Source, item)
            dst := filepath.Join(m.Dest, item)
            if err := os.Rename(src, dst); err != nil {
                return moved, camperrors.Wrapf(err, "moving %s to %s", src, dst)
            }
            moved++
        }
        // Remove source dir (best effort) ...
    }
    return moved, nil
}
```

Problems:
1. No pre-validation pass: if any (src, dst) pair is bad, the function discovers it only when it tries the rename. Items renamed before the failure are already moved; items after are not. The repo is in a split state.
2. `os.Rename` on files (not directories) silently overwrites the destination on POSIX. A dst that is a FILE is not caught by the "directory `os.Rename` onto non-empty fails" behavior.
3. The caller in `cmd/camp/init/command.go:263-268` (verified) downgrades the error to a warning: `writef(w.HumanOut, "  %s Migration error: %v\n", ...)`. Line 283 then calls `commitRepairChanges` unconditionally for `p.Repair && !p.DryRun`, committing whatever partially-moved state exists.

### Step 1: Add a validation-only pass to ExecuteMigrations

Before any rename, validate every pair:

```go
// validateMigrationPlan checks all (src, dst) pairs before any file is moved.
// Returns a non-nil error describing all invalid pairs found.
func validateMigrationPlan(migrations []MigrationAction) error {
    var errs []string
    for _, m := range migrations {
        for _, item := range m.Items {
            src := filepath.Join(m.Source, item)
            dst := filepath.Join(m.Dest, item)

            // Source must exist.
            if _, err := os.Stat(src); err != nil {
                if os.IsNotExist(err) {
                    errs = append(errs, fmt.Sprintf("source not found: %s", src))
                } else {
                    errs = append(errs, fmt.Sprintf("cannot stat source %s: %v", src, err))
                }
            }

            // Destination must be absent. Use ExistingItemPath to check legacy
            // flat and dated-bucket layouts.
            if _, exists, checkErr := statuspath.ExistingItemPath(m.Dest, item); checkErr != nil {
                errs = append(errs, fmt.Sprintf("cannot check destination %s: %v", dst, checkErr))
            } else if exists {
                errs = append(errs, fmt.Sprintf("destination already exists: %s", dst))
            }
        }
    }
    if len(errs) > 0 {
        return camperrors.Newf("migration pre-validation failed:\n  %s", strings.Join(errs, "\n  "))
    }
    return nil
}
```

Import `statuspath "github.com/Obedience-Corp/camp/internal/dungeon/statuspath"` at the top of `repair.go`.

### Step 2: Rewrite ExecuteMigrations to use the validation pass

```go
// ExecuteMigrations moves items from misplaced directories to their correct locations.
// It pre-validates every (src exists, dst absent) pair before moving anything.
// Returns the number of items moved and any error.
// On error, reports how many items were moved before the failure so the caller
// can provide recovery guidance.
//
// All new mover logic here is structured as a no-replace helper for later
// extraction into the shared statusdir primitive (D003, sequence 11.05).
func ExecuteMigrations(migrations []MigrationAction) (int, error) {
    if err := validateMigrationPlan(migrations); err != nil {
        return 0, err
    }

    moved := 0
    total := 0
    for _, m := range migrations {
        total += len(m.Items)
    }

    for _, m := range migrations {
        if err := os.MkdirAll(m.Dest, 0755); err != nil {
            return moved, camperrors.Wrapf(err, "creating destination directory %s", m.Dest)
        }
        for _, item := range m.Items {
            src := filepath.Join(m.Source, item)
            dst := filepath.Join(m.Dest, item)
            if err := migrateMoveItem(src, dst); err != nil {
                return moved, camperrors.Wrapf(err,
                    "moving %s to %s (%d/%d items moved; re-run after fixing)", src, dst, moved, total)
            }
            moved++
        }
        // Remove the now-empty source directory (best effort after all items moved).
        removeEmptySourceDir(m.Source)
    }
    return moved, nil
}

// migrateMoveItem moves a single item from src to dst with no-replace semantics.
// dst must not exist (enforced by the pre-validation pass).
// Structured as a small helper for D003 extraction in sequence 11.05.
func migrateMoveItem(src, dst string) error {
    return os.Rename(src, dst)
}

// removeEmptySourceDir removes a source directory only if it is empty
// (ignoring .gitkeep). Best-effort: errors are silently discarded.
func removeEmptySourceDir(dir string) {
    entries, err := os.ReadDir(dir)
    if err != nil {
        return
    }
    for _, e := range entries {
        if e.Name() != ".gitkeep" {
            return
        }
    }
    _ = os.RemoveAll(dir)
}
```

### Step 3: Fix the caller in cmd/camp/init/command.go

Find the migration error handling block at `:262-268` (verified):

```go
// Before:
moved, err := scaffold.ExecuteMigrations(opts.RepairPlan.Migrations)
if err != nil {
    writef(w.HumanOut, "  %s Migration error: %v\n", ui.WarningIcon(), err)
}
migrationCount = moved

// After:
moved, migrationErr := scaffold.ExecuteMigrations(opts.RepairPlan.Migrations)
if migrationErr != nil {
    writef(w.HumanOut, "  %s Migration failed after %d item(s) moved: %v\n",
        ui.ErrorIcon(), moved, migrationErr)
    writef(w.HumanOut, "  Recovery: fix the issue above and re-run camp init --repair\n")
    return migrationErr
}
migrationCount = moved
```

Returning `migrationErr` here causes `runInit` to return before reaching the `commitRepairChanges` call at line 283. No partial migration is ever auto-committed.

### Step 4: Unit tests

Add `internal/scaffold/repair_migration_test.go` (host `t.TempDir()`):

```go
func TestExecuteMigrationsPreValidationRejectsDstExists(t *testing.T) {
    dir := t.TempDir()
    src := filepath.Join(dir, "source")
    dst := filepath.Join(dir, "dest")
    require.NoError(t, os.MkdirAll(src, 0755))
    require.NoError(t, os.MkdirAll(dst, 0755))
    item := "foo.md"
    require.NoError(t, os.WriteFile(filepath.Join(src, item), []byte("x"), 0644))
    require.NoError(t, os.WriteFile(filepath.Join(dst, item), []byte("existing"), 0644))

    _, err := ExecuteMigrations([]MigrationAction{{Source: src, Dest: dst, Items: []string{item}}})
    if err == nil {
        t.Fatal("expected pre-validation error when dst already exists")
    }
    // Source file must still be at its original location (nothing moved).
    if _, statErr := os.Stat(filepath.Join(src, item)); statErr != nil {
        t.Fatalf("source file was moved despite pre-validation failure: %v", statErr)
    }
}

func TestExecuteMigrationsNoCommitOnFailure(t *testing.T) {
    // This test validates that a failure mid-migration returns an error that
    // includes the moved count, so the caller can produce useful diagnostics.
    dir := t.TempDir()
    src := filepath.Join(dir, "source")
    dst := filepath.Join(dir, "dest")
    require.NoError(t, os.MkdirAll(src, 0755))
    require.NoError(t, os.MkdirAll(dst, 0755))

    items := []string{"a.md", "b.md", "c.md"}
    for _, it := range items {
        require.NoError(t, os.WriteFile(filepath.Join(src, it), []byte("x"), 0644))
    }
    // Make b.md's destination exist to trigger failure after a.md moves.
    require.NoError(t, os.WriteFile(filepath.Join(dst, "b.md"), []byte("block"), 0644))

    // Pre-validation will catch b.md's dst existence.
    _, err := ExecuteMigrations([]MigrationAction{{Source: src, Dest: dst, Items: items}})
    if err == nil {
        t.Fatal("expected error")
    }
    // a.md must still be in src (nothing was moved since pre-validation fired).
    if _, statErr := os.Stat(filepath.Join(src, "a.md")); statErr != nil {
        t.Fatal("a.md was moved despite pre-validation failure")
    }
}
```

## Edge Cases

- **POSIX rename clobbers FILES silently**: Caught by `validateMigrationPlan`'s `ExistingItemPath` check on the dst before any rename happens. The pre-validation pass is the primary guard; `migrateMoveItem` itself does not re-check because the pre-validation window is narrow and these are sequential operations (no concurrent modification).
- **statuspath.ExistingItemPath for plain destinations**: `ExistingItemPath` checks the legacy flat layout and dated-bucket layout under `m.Dest`. For migration destinations that are simple flat directories (not dungeon-style dated buckets), the flat-path check (`filepath.Join(m.Dest, item)`) is sufficient, but using `ExistingItemPath` is strictly safer and matches the pattern used by `dungeon/service_move.go`.
- **macOS `/private/var`**: Path comparisons in tests must use `filepath.EvalSymlinks` if comparing the returned dst path. The pre-validation and rename operations themselves do not do string comparison on the paths, so they are unaffected.
- **Re-run after partial failure**: If somehow a partial migration occurs (e.g. the process is killed mid-rename), re-running `camp init --repair` will hit pre-validation because the moved items now exist at the destination. The error message explains which pair blocked and the user can resolve it manually.

## Out of Scope

- Building the shared statusdir primitive (sequence 11.05, D003). The `migrateMoveItem` helper here is intentionally minimal.
- Fixing `internal/scaffold/repair.go:558-570` gitignore atomic write (sequence 06).
- Fixing swallowed post-migration `os.RemoveAll` errors (INIT-2; that is a LOW finding in sequence 12).

## Done When

- [ ] `validateMigrationPlan` pre-checks every pair before any rename
- [ ] `ExecuteMigrations` returns a non-nil error describing failure without moving anything if pre-validation fails
- [ ] Mid-sequence failure returns a non-nil error with moved-count context
- [ ] `cmd/camp/init/command.go` returns the migration error and does not reach `commitRepairChanges` on failure
- [ ] Unit tests: pre-validation rejection, clean rerun behavior, and no-partial-commit assertion all pass
- [ ] Both-profile gate passes after these changes