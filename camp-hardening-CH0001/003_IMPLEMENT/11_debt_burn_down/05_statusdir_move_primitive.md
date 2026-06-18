---
fest_type: task
fest_id: 05_statusdir_move_primitive.md
fest_name: statusdir_move_primitive
fest_parent: 11_debt_burn_down
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.788007-06:00
fest_updated: 2026-06-15T07:24:39.83637-06:00
fest_tracking: true
---


# Task: Shared Statusdir Move Primitive

## Objective

Build one internal package owning the no-replace atomic move semantics and migrate all four parallel implementations into it, fixing the dungeon TOCTOU, stat-conflation, and missing collision tests as a side effect, per D003.

## Execution Notes

- Schema-driven workflow decision: keep the hardcoded dungeon taxonomy (`completed`, `archived`, `someday`) for this task. `internal/workflow` already provides schema-driven movement for workflow statuses, but dungeon scaffold/routing still owns terminal-status compatibility, docs routing, and dated bucket behavior. Replacing that taxonomy should be a separate compatibility migration after the schema model can express those dungeon-specific contracts.

## Requirements

- [ ] One package (`internal/statusmove` or extending `internal/workflow/statuspath`) owns: no-replace semantics, dated buckets, cross-device fallback, collision policy
- [ ] Darwin-specific rename semantics documented in the implementation (see Background)
- [ ] All four callers migrated: `internal/workflow/service_move.go`, `internal/dungeon/service_move.go` (four blocks plus `docs_routing.go`), `internal/intent/helpers.go moveFile`, `internal/scaffold/repair.go` and `internal/workflow/service_sync_migrate.go`
- [ ] Dungeon `service_move.go:29-31` stat-conflation fixed: any stat error that is NOT `os.IsNotExist` returns the real error, not `ErrNotFound`
- [ ] Dungeon TOCTOU eliminated by the primitive's no-replace semantics (no stat-then-rename window)
- [ ] `MoveToDungeon` gains `pathutil.ValidateBoundary` like its siblings
- [ ] Missing collision tests added for `MoveToDungeon` and `MoveToDocs`
- [ ] Sequence-03 regression tests (from `03_sync_and_migration_safety`) still pass against the new primitive (they are the conformance suite per D003)
- [ ] Interim helpers from sequence 03 deleted (replaced by this primitive)
- [ ] Decision on whether schema-driven workflow supersedes hardcoded dungeon recorded in task notes during execution
- [ ] Both-profile gate green after the migration commit

## Implementation

### Background

R25 and D003 document four parallel implementations of "move a markdown item between status directories":

1. `internal/workflow/service_move.go:44-59`: check then rename (collision check exists, TOCTOU window present)
2. `internal/dungeon/service_move.go` four blocks (`:16-46`, `:50-104`, `:105-148`, `:150-199`) plus `docs_routing.go:125-131`: stat-then-rename TOCTOU; `MoveToDungeon` at `:16-46` maps ANY stat error to `ErrNotFound` (conflation) and lacks `ValidateBoundary`; `MoveToDungeon` and `MoveToDocs` have no collision tests
3. `internal/intent/helpers.go:46` `moveFile`: cross-device fallback present but `os.Create` (O_TRUNC) on ANY rename error, not just EXDEV; sequence 03 task 06 hardened this into a small helper
4. `internal/scaffold/repair.go:666` and `internal/workflow/service_sync_migrate.go:185`: bare `os.Rename` with no destination check; sequence 03 task 04/05 added pre-validation helpers

D003 decided: sequence 03 fixes the P0 correctness bugs surgically; sequence 11 task 05 (this task) builds the shared primitive and migrates all four. The sequence-03 regression tests are the conformance suite.

**Darwin / macOS no-replace semantics:**
On Linux, `renameat2(RENAME_NOREPLACE)` provides atomic no-replace via a single syscall.
On macOS, there is no equivalent: `renameatx_np(RENAME_EXCL)` is an undocumented private API. The portable fallback is: `link(src, dst)` (fails with EEXIST if dst exists) then `unlink(src)` (if link succeeded). This is the `O_EXCL link-then-remove` strategy. Document this in the package with a build-tag constant for the darwin vs linux path.

### Step 1: Design the `internal/statusmove` package

Create `internal/statusmove/move.go`:

```go
package statusmove

import (
    "context"
    "os"
    "path/filepath"
    "time"

    "github.com/Obedience-Corp/camp/internal/pathutil"
    camperrors "github.com/Obedience-Corp/camp/internal/errors"
)

// MoveOptions configures a status-directory move.
type MoveOptions struct {
    // DatedBucket, if true, places dest inside a YYYY-MM-DD subdirectory.
    DatedBucket bool
    // BoundaryRoot, if non-empty, validates that dest remains under this root.
    BoundaryRoot string
    // Now overrides the current time for dated bucket naming (nil = time.Now).
    Now *time.Time
}

// Move moves src to dst atomically with no-replace semantics.
// If dst exists, returns ErrAlreadyExists immediately (no overwrite).
// Cross-device moves fall back to copy-then-delete.
// If opts.DatedBucket is true, dst is placed inside dst/YYYY-MM-DD/.
func Move(ctx context.Context, src, dst string, opts MoveOptions) error {
    if err := ctx.Err(); err != nil {
        return camperrors.Wrap(err, "context cancelled")
    }

    // Stat source (distinguish not-found from permission errors)
    if _, err := os.Stat(src); err != nil {
        if os.IsNotExist(err) {
            return camperrors.Wrap(camperrors.ErrNotFound, src)
        }
        return camperrors.Wrapf(err, "stat source %s", src)
    }

    // Apply dated bucket if requested
    if opts.DatedBucket {
        now := time.Now()
        if opts.Now != nil {
            now = *opts.Now
        }
        dst = filepath.Join(dst, now.Format("2006-01-02"), filepath.Base(src))
    }

    // Validate boundary
    if opts.BoundaryRoot != "" {
        if err := pathutil.ValidateBoundary(dst, opts.BoundaryRoot); err != nil {
            return camperrors.Wrapf(err, "move destination %s outside boundary", dst)
        }
    }

    // Ensure destination directory exists
    if err := os.MkdirAll(filepath.Dir(dst), 0755); err != nil {
        return camperrors.Wrapf(err, "create destination directory for %s", dst)
    }

    return noReplaceMove(src, dst)
}
```

Create `internal/statusmove/noreplace_linux.go` (build tag `linux`):
```go
//go:build linux

package statusmove

import "golang.org/x/sys/unix"

func noReplaceMove(src, dst string) error {
    // renameat2 with RENAME_NOREPLACE is atomic no-replace on Linux.
    err := unix.Renameat2(unix.AT_FDCWD, src, unix.AT_FDCWD, dst, unix.RENAME_NOREPLACE)
    if err == unix.EEXIST || err == unix.ENOTEMPTY {
        return ErrAlreadyExists
    }
    if err == unix.EXDEV {
        return crossDeviceMove(src, dst) // fallback defined below
    }
    return err
}
```

Create `internal/statusmove/noreplace_darwin.go` (build tag `darwin`):
```go
//go:build darwin

package statusmove

import (
    "os"
    "syscall"

    camperrors "github.com/Obedience-Corp/camp/internal/errors"
)

// noReplaceMove on Darwin uses link+unlink because renameatx_np is private API.
// link(src, dst) fails with EEXIST if dst already exists (atomic exclusivity).
// unlink(src) removes the original after the new name is established.
func noReplaceMove(src, dst string) error {
    err := os.Link(src, dst)
    if err != nil {
        if linkErr, ok := err.(*os.LinkError); ok {
            if linkErr.Err == syscall.EEXIST {
                return ErrAlreadyExists
            }
            if linkErr.Err == syscall.EXDEV {
                return crossDeviceMove(src, dst)
            }
        }
        return camperrors.Wrapf(err, "link %s -> %s", src, dst)
    }
    // Link succeeded; remove the original.
    if err := os.Remove(src); err != nil {
        // The link succeeded; the new name is established. Failure to remove
        // the old name leaves a duplicate but is not data-loss. Warn via error.
        return camperrors.Wrapf(err, "remove source after link %s", src)
    }
    return nil
}
```

Create `internal/statusmove/noreplace_other.go` (build tag `!linux,!darwin`):
```go
//go:build !linux && !darwin

package statusmove

// noReplaceMove on unsupported platforms uses a best-effort approach.
// There is no atomic no-replace rename; the window is documented.
func noReplaceMove(src, dst string) error {
    if _, err := os.Stat(dst); err == nil {
        return ErrAlreadyExists
    }
    return crossDeviceMove(src, dst) // uses os.Rename with EXDEV fallback
}
```

Create `internal/statusmove/crossdevice.go`:
```go
package statusmove

import (
    "io"
    "os"
    "syscall"
    camperrors "github.com/Obedience-Corp/camp/internal/errors"
)

var ErrAlreadyExists = camperrors.Wrap(camperrors.ErrAlreadyExists, "statusmove: destination already exists")

func crossDeviceMove(src, dst string) error {
    err := os.Rename(src, dst)
    if err == nil {
        return nil
    }
    // Check for EXDEV (cross-device rename)
    if linkErr, ok := err.(*os.LinkError); ok && linkErr.Err == syscall.EXDEV {
        return copyThenDelete(src, dst)
    }
    return camperrors.Wrapf(err, "rename %s -> %s", src, dst)
}

func copyThenDelete(src, dst string) error {
    in, err := os.Open(src)
    if err != nil {
        return camperrors.Wrapf(err, "open %s", src)
    }
    defer in.Close()
    out, err := os.OpenFile(dst, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
    if err != nil {
        if os.IsExist(err) {
            return ErrAlreadyExists
        }
        return camperrors.Wrapf(err, "create %s", dst)
    }
    defer out.Close()
    if _, err := io.Copy(out, in); err != nil {
        _ = os.Remove(dst)
        return camperrors.Wrapf(err, "copy %s -> %s", src, dst)
    }
    if err := out.Sync(); err != nil {
        _ = os.Remove(dst)
        return camperrors.Wrapf(err, "sync %s", dst)
    }
    return os.Remove(src)
}
```

Note: if `golang.org/x/sys/unix` is not already a dependency, check `go.mod`. If absent, the Linux file can fall back to a syscall-based approach without the external package, or use a simpler build-constrained implementation using `os.Link`.

### Step 2: Migrate `internal/workflow/service_move.go`

The current destination check (`resolveWorkflowItemPath`) is at `:44-59`. Replace the check-then-rename with a call to `statusmove.Move`:

```go
import "github.com/Obedience-Corp/camp/internal/statusmove"

// In Move:
if err := statusmove.Move(ctx, itemPath, destPath, statusmove.MoveOptions{}); err != nil {
    if errors.Is(err, statusmove.ErrAlreadyExists) {
        return nil, camperrors.Wrap(ErrAlreadyExists, destPath)
    }
    return nil, camperrors.Wrap(err, "failed to move item")
}
```

The sequence-03 interim helper (if any was added in `service_move.go`) is replaced and deleted.

### Step 3: Migrate `internal/dungeon/service_move.go`

Four blocks plus `docs_routing.go:125-131`. For each block:

1. **Fix stat-conflation in `MoveToDungeon`**: the current `:29-31` maps any stat error to `ErrNotFound`. Replace with `os.IsNotExist`:

   ```go
   if _, err := os.Stat(sourcePath); err != nil {
       if os.IsNotExist(err) {
           return camperrors.Wrap(ErrNotFound, itemName)
       }
       return camperrors.Wrapf(err, "stat source %s", sourcePath)
   }
   ```

2. **Add `ValidateBoundary` to `MoveToDungeon`**: after resolving `targetPath`, call `pathutil.ValidateBoundary(targetPath, s.campaignRoot)`.

3. **Replace all four stat-then-rename blocks with `statusmove.Move`**:

   ```go
   if err := statusmove.Move(ctx, sourcePath, targetPath, statusmove.MoveOptions{
       BoundaryRoot: s.campaignRoot,
   }); err != nil {
       if errors.Is(err, statusmove.ErrAlreadyExists) {
           return camperrors.Wrapf(ErrAlreadyExists, ...)
       }
       return camperrors.Wrapf(err, ...)
   }
   ```

4. **Add missing collision tests** for `MoveToDungeon` and `MoveToDocs` in `service_move_test.go`:

   ```go
   func TestMoveToDungeonCollision(t *testing.T) {
       // Create a dungeon dir with itemName already present
       // Expect ErrAlreadyExists on second move
   }
   func TestMoveToDocsCollision(t *testing.T) {
       // Same pattern
   }
   ```

### Step 4: Migrate `internal/intent/helpers.go moveFile`

The sequence-03 hardened version of `moveFile` is the EXDEV-only fallback. Replace the entire `moveFile` with a delegation to `statusmove`:

```go
func moveFile(src, dst string) error {
    return statusmove.Move(context.Background(), src, dst, statusmove.MoveOptions{})
}
```

If sequence 03 created an interim helper with no-replace semantics, verify it is subsumed and delete it.

### Step 5: Migrate `internal/scaffold/repair.go` and `internal/workflow/service_sync_migrate.go`

The sequence-03 surgical fixes added pre-validation via `statuspath.ExistingItemPath`. Replace those with `statusmove.Move` which provides the same guarantee atomically:

```go
// In repair.go ExecuteMigrations:
if err := statusmove.Move(ctx, src, dst, statusmove.MoveOptions{}); err != nil {
    if errors.Is(err, statusmove.ErrAlreadyExists) {
        return camperrors.Wrapf(ErrMigrationConflict, "%s already exists", dst)
    }
    return camperrors.Wrapf(err, "migrate %s to %s", src, dst)
}
```

### Step 6: Schema-driven workflow decision

During execution, read the current state of `internal/workflow/service_move.go` and `internal/dungeon/scaffold`. The question: should the hardcoded `dungeon/{completed,archived,someday}` taxonomy be superseded by schema-driven workflow (`internal/workflow`)?

Record the answer in the task notes (a few sentences in the .fest/metrics or a comment in the commit). Do not pre-decide here; D003 deferred this to execution time.

## Done When

- [ ] `internal/statusmove` package exists with Linux, Darwin, and fallback variants
- [ ] `go test ./internal/statusmove/...` passes including the collision test
- [ ] `grep -rn "os.Rename" internal/workflow/service_move.go internal/dungeon/service_move.go internal/intent/helpers.go` returns 0 non-EXDEV-fallback results
- [ ] `MoveToDungeon` calls `pathutil.ValidateBoundary`
- [ ] `MoveToDungeon` and `MoveToDocs` have collision tests
- [ ] `service_move.go:29-31` stat-conflation fixed (permission error returns the real error)
- [ ] Sequence-03 interim helpers deleted
- [ ] Both-profile gate green
- [ ] Schema-driven workflow decision recorded in task notes