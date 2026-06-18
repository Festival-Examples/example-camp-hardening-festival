---
fest_type: task
fest_id: 02_cp_copy_guards.md
fest_name: cp_copy_guards
fest_parent: 02_destructive_command_safety
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.61963-06:00
fest_updated: 2026-06-12T13:51:41.758612-06:00
fest_tracking: true
---


# Task: cp/copy Guards

## Objective

Prevent `camp cp --force` from truncating a source file to zero bytes when the resolved destination is the same inode, and prevent directory copies where the destination is inside the source tree, by adding `os.SameFile` and dest-under-src guards before opening any file for writing (N-1).

## Requirements

- [ ] `runCopy` in `cmd/camp/copy.go` detects when the resolved `src` and `dest` paths refer to the same filesystem inode (via `os.SameFile`) and returns an error with the message `"source and destination are the same file"` before opening the destination for writing.
- [ ] For directory copies, `runCopy` detects when `dest` resolves to a path under `src` (after `filepath.EvalSymlinks` on both, with separator-guarded prefix check to handle macOS `/var` vs `/private/var`) and returns an error with the message `"cannot copy a directory into itself"`.
- [ ] The non-force error message is updated from `"destination %q already exists (use --force to overwrite)"` to `"destination %q already exists"` to stop steering users into the destructive `--force` invocation.
- [ ] The guards apply regardless of whether `--force` is passed; `--force` controls destination-exists behavior only, not same-file or self-recursion behavior.
- [ ] `transfer.CopyFile` and `transfer.CopyDir` in `internal/transfer/copy.go` are not modified for the guard logic; the checks belong in `runCopy` where both src and dest are known before any open occurs.
- [ ] Container integration tests in `tests/integration/cp_copy_guards_test.go` cover: same-file truncation is prevented, self-recursion is prevented, and normal file/directory copies still work.

## Implementation

### Background: the bug mechanics

`internal/transfer/copy.go:24` opens the destination with `os.OpenFile(dest, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, ...)`. The `O_TRUNC` flag truncates the file to zero length at open time, before any byte is written. If `src` and `dest` resolve to the same inode, the source file is emptied before the copy reads a single byte.

The call sequence that causes this from `cmd/camp/copy.go:75-101` (worktree line numbers verified):
1. `dest = filepath.Join(dest, filepath.Base(src))` when dest is a directory (line 76) -- this is how `camp cp notes.md .` from within the notes.md directory becomes `notes.md/notes.md` then resolves back to `notes.md`.
2. `--force` skips the `os.Stat(dest)` existence check at lines 80-83.
3. `transfer.CopyFile(src, dest)` opens dest with `O_TRUNC` (line 98-99 in copy.go, actual open at `internal/transfer/copy.go:24`).

For directories, `transfer.CopyDir` at `internal/transfer/copy.go:46` calls `os.MkdirAll(dest, ...)` before walking `src`. When dest is inside src, `filepath.WalkDir` descends into dest and finds its own just-created output, recurse indefinitely (or until disk full).

The non-force error message at `cmd/camp/copy.go:82` currently reads:
```
destination %q already exists (use --force to overwrite)
```
This actively guides the user toward `--force`, which is precisely the flag that triggers the truncation. The fix removes the parenthetical.

### Step 1: Add same-file guard in `runCopy`

Insert a same-file check in `cmd/camp/copy.go` after the existing src validation (after line 72, before the dest-is-dir join at line 75). The check uses `os.Stat` + `os.SameFile`:

```go
// Stat src for the same-file check. The ValidatePathExists call above
// already verified src exists, so a Stat failure here is unexpected.
srcStatForSame, err := os.Stat(src)
if err != nil {
    return camperrors.Wrap(err, "stat source")
}
```

After the dest-is-dir join and shortcut resolution (after the `filepath.Join(dest, filepath.Base(src))` line), add:

```go
// Same-file check: stat the resolved destination. If it exists and refers
// to the same inode as src, refuse before opening with O_TRUNC.
if destStatForSame, err := os.Stat(dest); err == nil {
    if os.SameFile(srcStatForSame, destStatForSame) {
        return fmt.Errorf("source and destination are the same file: %s", dest)
    }
}
```

This runs for both file and directory cases. For directories the SameFile check catches the top-level identity; the dest-under-src check below catches subdirectory recursion.

### Step 2: Add dest-under-src guard for directory copies

After the same-file check, add a guard specifically for directory sources. Insert before the `if !force` block (before line 79 in copy.go):

```go
if srcInfo.IsDir() {
    // Resolve both paths to their canonical forms before comparing.
    // On macOS, /var is a symlink to /private/var; without EvalSymlinks
    // a prefix check would silently fail and allow the recursion.
    resolvedSrc, err := filepath.EvalSymlinks(src)
    if err != nil {
        return camperrors.Wrap(err, "resolve source path")
    }
    resolvedDest, resolvedDestErr := filepath.EvalSymlinks(dest)
    if resolvedDestErr != nil {
        // dest may not exist yet; resolve the nearest existing ancestor.
        // Use pathutil.ValidateBoundary for this (already in the codebase).
        // If it doesn't exist, self-recursion is impossible.
        resolvedDest = dest
    }
    // Guard: dest must not be inside src. Use separator-guarded prefix check
    // so that /foo/bar does not falsely match /foo/barsuffix.
    srcWithSep := resolvedSrc + string(filepath.Separator)
    if resolvedDest == resolvedSrc || strings.HasPrefix(resolvedDest, srcWithSep) {
        return fmt.Errorf("cannot copy a directory into itself: %s is inside %s", dest, src)
    }
}
```

Add `"strings"` to the import block if not already present.

Note: this block reads `srcInfo.IsDir()` which requires moving the `srcInfo` stat before the guard. In the current code, `srcInfo` is obtained at line 85 (after the `if !force` block). Restructure to stat src once before both guards:

```go
// Stat source (needed for both directory guards and the copy dispatch below).
srcInfo, err := os.Stat(src)
if err != nil {
    return camperrors.Wrap(err, "stat source")
}
```

Place this immediately after `ValidatePathExists(src)` returns nil. Remove the duplicate `srcInfo` stat at line 85.

### Step 3: Fix the non-force error message

Change line 82 of `cmd/camp/copy.go` from:

```go
return fmt.Errorf("destination %q already exists (use --force to overwrite)", dest)
```

to:

```go
return fmt.Errorf("destination %q already exists", dest)
```

The `--force` flag is documented in the help text; the error message should not suggest destructive escalation.

### Final structure of `runCopy`

The key changes produce this flow in `runCopy`:

1. Resolve src and dest paths (existing shortcut logic).
2. `ValidatePathExists(src)` (existing).
3. Dest-is-dir join: `dest = filepath.Join(dest, filepath.Base(src))` when dest is a dir (existing).
4. `srcInfo, err := os.Stat(src)` -- moved up.
5. Same-file check (new): stat dest, call `os.SameFile`, refuse if true.
6. Dest-under-src check (new): for `srcInfo.IsDir()`, EvalSymlinks both, separator-guarded prefix check.
7. Non-force existence check (existing, message fixed).
8. Copy dispatch (existing).

### Edge cases

**macOS `/var` symlink:** On macOS, `/var/folders/.../myfile` resolves to `/private/var/folders/.../myfile`. `filepath.EvalSymlinks` handles this correctly. The `os.SameFile` check uses the `os.FileInfo` structs returned by `os.Stat`, which follows symlinks, so SameFile works correctly even on macOS without manual EvalSymlinks. The explicit EvalSymlinks in step 2 is only needed for the string-based prefix comparison.

**Destination does not exist yet:** `os.Stat(dest)` returns an error; the same-file check is skipped (destination cannot be the same inode if it doesn't exist). The dest-under-src check for a non-existent dest does not need EvalSymlinks on dest; a non-existent path cannot be inside an existing src unless its parent is already inside src, and `strings.HasPrefix` on the raw (cleaned) path is sufficient for that case.

**Symlinked source or destination:** `os.Stat` follows symlinks, so `os.SameFile` correctly compares the underlying inodes even when src or dest is a symlink. EvalSymlinks in the dest-under-src check ensures the resolved canonical form is compared.

### Container integration tests

Create `tests/integration/cp_copy_guards_test.go`:

```go
//go:build integration
// +build integration

package integration

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestCp_SameFileGuard(t *testing.T) {
    tc := GetSharedContainer(t)

    _, err := tc.InitCampaign("/campaigns/cp-samefile", "cp-samefile", "product")
    require.NoError(t, err)

    // Create a file with known content.
    tc.Shell(t, `
set -e
mkdir -p /campaigns/cp-samefile/docs
printf 'important content\n' > /campaigns/cp-samefile/docs/notes.md
`)

    // Try to copy the file into its own directory (same-file truncation scenario).
    _, err = tc.RunCampInDir("/campaigns/cp-samefile/docs",
        "cp", "--force", "notes.md", ".")
    assert.Error(t, err, "same-file copy should fail")
    assert.Contains(t, err.Error(), "same file", "error should mention same file")

    // Verify the original file content is unchanged.
    content, err := tc.ReadFile("/campaigns/cp-samefile/docs/notes.md")
    require.NoError(t, err)
    assert.Contains(t, content, "important content", "file content must not be truncated")
}

func TestCp_SelfRecursionGuard(t *testing.T) {
    tc := GetSharedContainer(t)

    _, err := tc.InitCampaign("/campaigns/cp-recurse", "cp-recurse", "product")
    require.NoError(t, err)

    tc.Shell(t, `
set -e
mkdir -p /campaigns/cp-recurse/mydir
printf 'file1\n' > /campaigns/cp-recurse/mydir/a.txt
printf 'file2\n' > /campaigns/cp-recurse/mydir/b.txt
`)

    // Try to copy a directory into itself.
    _, err = tc.RunCampInDir("/campaigns/cp-recurse",
        "cp", "--force", "mydir", "mydir/subdir")
    assert.Error(t, err, "copying a directory into itself should fail")
    assert.Contains(t, err.Error(), "into itself", "error should mention self-copy")

    // Original files must still exist.
    exists, err := tc.CheckFileExists("/campaigns/cp-recurse/mydir/a.txt")
    require.NoError(t, err)
    assert.True(t, exists, "original files must not be affected")
}

func TestCp_NormalFileCopyWorks(t *testing.T) {
    tc := GetSharedContainer(t)

    _, err := tc.InitCampaign("/campaigns/cp-normal", "cp-normal", "product")
    require.NoError(t, err)

    tc.Shell(t, `
set -e
mkdir -p /campaigns/cp-normal/src /campaigns/cp-normal/dest
printf 'hello world\n' > /campaigns/cp-normal/src/file.md
`)

    _, err = tc.RunCampInDir("/campaigns/cp-normal",
        "cp", "src/file.md", "dest/")
    require.NoError(t, err, "normal file copy should succeed")

    content, err := tc.ReadFile("/campaigns/cp-normal/dest/file.md")
    require.NoError(t, err)
    assert.Contains(t, content, "hello world")
}

func TestCp_NormalDirCopyWorks(t *testing.T) {
    tc := GetSharedContainer(t)

    _, err := tc.InitCampaign("/campaigns/cp-dir", "cp-dir", "product")
    require.NoError(t, err)

    tc.Shell(t, `
set -e
mkdir -p /campaigns/cp-dir/srcdir
printf 'alpha\n' > /campaigns/cp-dir/srcdir/alpha.md
printf 'beta\n'  > /campaigns/cp-dir/srcdir/beta.md
mkdir -p /campaigns/cp-dir/destparent
`)

    _, err = tc.RunCampInDir("/campaigns/cp-dir",
        "cp", "srcdir", "destparent/")
    require.NoError(t, err, "normal directory copy should succeed")

    content, err := tc.ReadFile("/campaigns/cp-dir/destparent/srcdir/alpha.md")
    require.NoError(t, err)
    assert.Contains(t, content, "alpha")
}
```

## Done When

- [ ] All requirements met
- [ ] `just build` and `BUILD_TAGS=dev just build` pass from the worktree root
- [ ] `just test unit` and `go test -tags dev ./...` pass
- [ ] `just lint` and `BUILD_TAGS=dev just lint` pass
- [ ] `just test integration` passes with the new `TestCp_*` tests
- [ ] Manual verification: `camp cp --force notes.md .` from the file's own directory returns an error and the file content is unchanged
- [ ] Manual verification: `camp cp --force mydir mydir/sub` returns an error

## Out of Scope

- `transfer.CopyFile` and `transfer.CopyDir` internals beyond the guard placement described here; the guard belongs at the `runCopy` layer where both src and dest are fully resolved.
- The `O_TRUNC` flag itself in `transfer.CopyFile`; it is correct behavior for normal overwrites and must not be changed.
- The `camp move` command, which is a separate command and verified clean by the second pass.