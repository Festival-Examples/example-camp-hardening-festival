---
fest_type: task
fest_id: 06_intent_movefile_and_promote_rollback.md
fest_name: intent_movefile_and_promote_rollback
fest_parent: 03_sync_and_migration_safety
fest_order: 6
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.638456-06:00
fest_updated: 2026-06-14T20:31:33.646464-06:00
fest_tracking: true
---


# Task: Guard moveFile against collisions and gate the copy fallback on EXDEV; fix promote rollback

**Finding:** R10/WF-5, N-19, N-12

## Objective

Add an ErrFileExists guard inside `moveFile` using O_CREATE|O_EXCL on the fallback open plus a stat guard before the rename, gate the copy-fallback on `errors.Is(err, syscall.EXDEV)` only, and fix `promoteToDesign`'s rollback to track creation with `os.Mkdir` so it removes only paths it created.

## Requirements

- [ ] `moveFile` in `internal/intent/helpers.go` stat-guards the destination before attempting `os.Rename`; if the destination exists, it returns `ErrFileExists` (or a wrapped sentinel) without touching either file.
- [ ] The fallback copy path in `moveFile` opens the destination with `os.OpenFile(dst, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)` so a concurrent or race-window creation of dst is caught rather than truncated.
- [ ] The copy fallback fires ONLY when the rename error wraps `syscall.EXDEV`. Any other rename error (permissions, path-not-found, etc.) is returned directly without opening the destination. Portable detection: `errors.Is(err, syscall.EXDEV)` with `*os.LinkError` unwrapping.
- [ ] Both call sites are covered: `CreateWithEditor` (`:173`) and `EditWithEditor`'s status-change path (`:226`) both call `moveFile`; the guard in `moveFile` itself covers both without per-call-site changes.
- [ ] `createDesignDoc` in `internal/intent/promote/promote.go` uses `os.Mkdir` (not `MkdirAll`) to create the design directory so that `createdNow` correctly reflects whether this invocation created the directory (not whether a parent existed).
- [ ] `removeCreatedDesignDoc` removes only `absDir` when `createdNow` is true; the check is already present in `promoteToDesign` at `:155-156` and `:165-166`. No behavior change is needed there IF `createdNow` is accurate; accuracy depends on the `os.Mkdir` fix in `createDesignDoc`.
- [ ] Unit tests: edited-id collision preserves the existing intent file; non-EXDEV rename error (e.g. simulated with a bad path) does not open the destination; failed promote with a pre-existing design dir leaves the user's files intact.

## Implementation

### Background

`moveFile` at `internal/intent/helpers.go:45-77` (worktree bc2ad1f, verified at `:46-77`):

```go
func moveFile(src, dst string) error {
    // Try rename first (same filesystem)
    if err := os.Rename(src, dst); err == nil {
        return nil
    }

    // Fall back to copy + delete (cross-device)
    srcFile, err := os.Open(src)
    if err != nil { ... }
    defer srcFile.Close()

    dstFile, err := os.Create(dst)   // O_TRUNC: truncates dst if it exists
    ...
}
```

Two problems:
1. `os.Rename(src, dst)` silently overwrites `dst` if it is a file on the same filesystem. No existence check.
2. The fallback `os.Create(dst)` (which is `O_CREATE|O_WRONLY|O_TRUNC`) fires on ANY rename error, not just EXDEV. A permission error, a missing parent, or a race window all route into the fallback, which then truncates the destination to zero.

`CreateWithEditor` calls `moveFile(tmpPath, finalPath)` at `:173` (verified) without any stat guard. If the user edits the `id:` field to match an existing intent's ID, `finalPath` already exists and `os.Rename` clobbers it. The second call site: `EditWithEditor`'s status-change block at `:226` calls `moveFile(originalPath, newPath)` directly (verified as the `moveFile` call in that function).

`createDesignDoc` at `promote.go:183-235` calls `os.MkdirAll(absDir, 0755)` at `:201`. `MkdirAll` returns nil if the directory already exists, so `createdNow` (the second return value) is determined only by whether `README.md` existed at `:223`. If `workflow/design/<slug>/` pre-exists with other files but no README, `createdNow` is set to true, and a later `Move` or `Save` failure triggers `removeCreatedDesignDoc` which calls `os.RemoveAll(absDir)`, destroying the pre-existing files.

### Step 1: Fix moveFile

Replace the body of `moveFile` at `internal/intent/helpers.go:46-77`:

```go
// moveFile moves src to dst with no-replace semantics.
// If dst already exists, it returns ErrFileExists without touching either file.
// The cross-device copy fallback fires ONLY on syscall.EXDEV.
// Structured as a no-replace helper for D003 extraction in sequence 11.05.
func moveFile(src, dst string) error {
    // Guard: refuse to clobber an existing destination.
    if _, err := os.Stat(dst); err == nil {
        return ErrFileExists
    } else if !os.IsNotExist(err) {
        return camperrors.Wrap(err, "checking destination")
    }

    // Fast path: same-filesystem rename.
    if err := os.Rename(src, dst); err == nil {
        return nil
    } else if !isExdevError(err) {
        // Any error other than EXDEV is returned as-is; do not open dst.
        return camperrors.Wrap(err, "renaming file")
    }

    // Cross-device fallback: copy + delete.
    // Use O_EXCL so a concurrent creation of dst is caught, not truncated.
    srcFile, err := os.Open(src)
    if err != nil {
        return camperrors.Wrap(err, "opening source file")
    }
    defer srcFile.Close()

    dstFile, err := os.OpenFile(dst, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
    if err != nil {
        if os.IsExist(err) {
            return ErrFileExists
        }
        return camperrors.Wrap(err, "creating destination file")
    }
    defer func() {
        dstFile.Close()
        // On any copy error, remove the partially-written dst so the caller
        // can retry without hitting a zero-byte file.
    }()

    if _, err := io.Copy(dstFile, srcFile); err != nil {
        _ = os.Remove(dst)
        return camperrors.Wrap(err, "copying file")
    }

    if err := dstFile.Sync(); err != nil {
        _ = os.Remove(dst)
        return camperrors.Wrap(err, "syncing destination file")
    }
    dstFile.Close()

    return os.Remove(src)
}

// isExdevError reports whether err is a cross-device rename error (syscall.EXDEV).
// On both Linux and macOS, os.Rename returns *os.LinkError wrapping syscall.EXDEV
// when src and dst are on different filesystems.
func isExdevError(err error) bool {
    var linkErr *os.LinkError
    if errors.As(err, &linkErr) {
        return errors.Is(linkErr.Err, syscall.EXDEV)
    }
    return errors.Is(err, syscall.EXDEV)
}
```

Add `"errors"` and `"syscall"` to the import block in `helpers.go`.

`ErrFileExists` must be defined in the `intent` package (or reused if it already exists). Check `internal/intent/errors.go`; if no `ErrFileExists` sentinel is present, add one:

```go
// ErrFileExists is returned when a move or create would overwrite an existing file.
var ErrFileExists = camperrors.New("file already exists at destination")
```

### Step 2: Fix createDesignDoc to use os.Mkdir

In `internal/intent/promote/promote.go:201`, replace `os.MkdirAll` with `os.Mkdir`:

```go
// Before:
if err := os.MkdirAll(absDir, 0755); err != nil {
    return "", false, camperrors.Wrap(err, "creating design directory")
}

// After:
if err := os.Mkdir(absDir, 0755); err != nil {
    if os.IsExist(err) {
        // Directory pre-exists; we did not create it.
        // The README check below will decide whether to proceed or return false.
        // createdNow will be false, so rollback will not remove this dir.
    } else {
        return "", false, camperrors.Wrap(err, "creating design directory")
    }
}
createdNow := true
if _, statErr := os.Stat(absDir); statErr == nil {
    // ...
}
```

Wait, a cleaner approach is to return `createdNow` based on whether `os.Mkdir` succeeded (not whether the directory already existed):

```go
mkdirErr := os.Mkdir(absDir, 0755)
var thisCallCreated bool
if mkdirErr == nil {
    thisCallCreated = true
} else if !os.IsExist(mkdirErr) {
    return "", false, camperrors.Wrap(mkdirErr, "creating design directory")
}
// thisCallCreated is false if the dir pre-existed; rollback must not remove it.
```

Then replace the `return relDir, true, nil` at the README-already-exists early return (`:225`) with `return relDir, false, nil` since we did not create the directory in that case either (if README exists, the dir pre-existed with content).

At the end of the function, return `thisCallCreated` instead of the hard-coded `true`:

```go
// line ~234 in worktree:
return relDir, thisCallCreated, nil
```

This ensures `createdNow` is true ONLY when THIS invocation called `os.Mkdir` successfully, which is exactly what `removeCreatedDesignDoc` should check before running `os.RemoveAll`.

The parent directories (`workflow/design/`) may need `os.MkdirAll` first:

```go
if err := os.MkdirAll(filepath.Dir(absDir), 0755); err != nil {
    return "", false, camperrors.Wrap(err, "creating design parent directory")
}
mkdirErr := os.Mkdir(absDir, 0755)
```

### Step 3: Unit tests

Add `internal/intent/helpers_test.go` (host `t.TempDir()`):

```go
func TestMoveFileDestExists_ReturnsErrFileExists(t *testing.T) {
    dir := t.TempDir()
    src := filepath.Join(dir, "src.md")
    dst := filepath.Join(dir, "dst.md")
    require.NoError(t, os.WriteFile(src, []byte("original"), 0644))
    require.NoError(t, os.WriteFile(dst, []byte("existing"), 0644))

    err := moveFile(src, dst)
    if !errors.Is(err, ErrFileExists) {
        t.Fatalf("expected ErrFileExists, got: %v", err)
    }
    // Both files must be unchanged.
    assertFileContent(t, src, "original")
    assertFileContent(t, dst, "existing")
}

func TestMoveFileNonExdevRenameError_DoesNotOpenDst(t *testing.T) {
    // Simulate a rename error that is NOT EXDEV (use a non-existent src dir).
    dir := t.TempDir()
    src := filepath.Join(dir, "nonexistent", "src.md")
    dst := filepath.Join(dir, "dst.md")

    err := moveFile(src, dst)
    if err == nil {
        t.Fatal("expected error for missing source dir")
    }
    // dst must NOT have been created (fallback must not have fired).
    if _, statErr := os.Stat(dst); !os.IsNotExist(statErr) {
        t.Fatal("dst was created despite non-EXDEV rename error")
    }
}
```

Add `internal/intent/promote/promote_rollback_test.go` (host `t.TempDir()`):

```go
func TestPromoteToDesignRollbackPreservesPreExistingDir(t *testing.T) {
    // Setup: workflow/design/<slug>/ already exists with user files but no README.
    dir := t.TempDir()
    designDir := filepath.Join(dir, "workflow", "design", "my-feature")
    require.NoError(t, os.MkdirAll(designDir, 0755))
    userFile := filepath.Join(designDir, "notes.md")
    require.NoError(t, os.WriteFile(userFile, []byte("my notes"), 0644))

    // The intent service and svc.Move will fail because we use a stub that
    // errors on Move. createDesignDoc uses os.Mkdir, which fails since the
    // dir pre-exists, so createdNow is false.
    // The rollback must NOT remove designDir.

    // ... construct a minimal stub intent service that returns an error on Move.
    // After the call, assert userFile still exists.
    assertFileContent(t, userFile, "my notes")
}
```

## Edge Cases

- **EXDEV detection on macOS vs Linux**: `os.Rename` across filesystems returns `*os.LinkError{Err: syscall.EXDEV}` on both platforms. `isExdevError` unwraps `*os.LinkError` first then falls back to direct `errors.Is`. This handles both the wrapped and unwrapped forms. On macOS under `/tmp` vs `/Volumes`, the test can trigger real EXDEV with a tmpdir on one mount and a destination on another; the container test environment has a single filesystem so EXDEV must be simulated (or tested separately in a multi-mount fixture if needed).
- **Race between stat-guard and rename**: The stat check before rename is a TOCTOU window. A concurrent writer can create `dst` between the Stat and the Rename. The Rename will fail because the dst now exists (same-filesystem POSIX rename does overwrite files but not directories; if dst is a file it DOES get clobbered). The O_EXCL fallback only fires on EXDEV. So the TOCTOU on Rename for files is a known limitation; document it in the function comment with the caveat "stat-guard before rename reduces the window but does not eliminate the TOCTOU for file destinations."
- **`ErrFileExists` vs callers that already handle `moveFile` errors**: Both `CreateWithEditor` and `EditWithEditor` return `moveFile`'s error wrapped with context. The callers' error messages already say "moving intent file" or "moving intent to new status"; wrapping `ErrFileExists` inside that context is fine. The service layer tests check for `ErrFileExists` presence via `errors.Is`.

## Out of Scope

- Converting `service.go` intent writes to atomic (sequence 06).
- The `O_CREATE|O_EXCL` collision guard on `CreateDirect` (already present in HEAD per the second-pass verified-negatives C-3).
- Fixing `moveFile`'s behavior for the `intent gather` flow or other callers outside `service.go` (those are tracked separately in R33/sequence 11).
- Building the shared statusdir primitive (sequence 11.05, D003).

## Verified Negatives (C-3, do not re-fix)

Intent status moves via `Move`/`UpdateDirect`/`Edit` are already collision-safe (verified in the second pass: `collisionSafeMovePath` and `pathAvailableForID` with regression tests). The guard added here targets `moveFile` specifically, which is the unguarded path used by `CreateWithEditor` and `EditWithEditor`'s status-change branch.

## Done When

- [ ] `moveFile` stat-guards dst before rename; returns `ErrFileExists` if dst exists
- [ ] `moveFile` copy fallback fires only on `isExdevError`; non-EXDEV errors return without opening dst
- [ ] `createDesignDoc` uses `os.Mkdir` (not MkdirAll); returns `createdNow = true` only when this call created the directory
- [ ] Unit tests pass: edited-id collision preserves existing intent; non-EXDEV error does not create dst; pre-existing design dir survives failed promote
- [ ] Both-profile gate passes after these changes