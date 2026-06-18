---
fest_type: task
fest_id: 01_intent_quest_atomic_writes.md
fest_name: intent_quest_atomic_writes
fest_parent: 06_atomicity_and_locking
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.695155-06:00
fest_updated: 2026-06-14T22:17:40.590183-06:00
fest_tracking: true
---


# Task: Intent and Quest Atomic Writes (WF-6 part 1)

## Objective

Convert every `os.WriteFile` call in the intent service and quest package to `fsutil.WriteFileAtomically` so that a crash or power loss mid-write cannot leave a half-written file where the original no longer exists.

## Requirements

- [x] **R11-WF6a**: `internal/intent/service.go:90` (`CreateDirect`) converted to `fsutil.WriteFileAtomically`.
- [x] **R11-WF6b**: `internal/intent/service.go:258` (`Save`, the in-place rewrite of an existing intent) converted.
- [x] **R11-WF6c**: `internal/intent/service.go:317` (the rename-then-write path, `MoveStatus`) converted.
- [x] **R11-WF6d**: Quest write `internal/quest/quest.go:271` (`save`) converted.
- [x] File modes preserved: `fsutil.WriteFileAtomically` already stat-reads the existing mode if the file exists and falls back to `defaultMode` for new files; pass `0644` as `defaultMode` at every call site to match the existing behavior.
- [x] Read-modify-write sites that need sequence-06 locking flagged with a `TODO(seq06-lock)` comment if not trivially lockable here; do not introduce locks in this task.
- [x] Existing test suites (`internal/intent/service_test.go`, `internal/quest/`) remain green in both profiles after the change.
- [x] New focused tests added: content integrity after a successful write; partial-write safety demonstrated by injecting a bad-directory condition (see Implementation section).

## Implementation

### Background: why atomic writes matter here

When camp or festival-app shells `camp intent` commands in parallel, or when a user runs two camp processes at once (common in agent loops), a plain `os.WriteFile` can lose half its write to a kernel preemption. The write goes to the same inode in place: there is no safe recovery copy. `fsutil.WriteFileAtomically` (at `internal/fsutil/atomic_write.go:14-48`) writes to a same-directory temp file, fsyncs, then renames over the destination. The rename is atomic at the POSIX level; readers always see either the old or the new content, never a partial mix. If the process dies between the write and the rename, the temp file is left behind (cleaned up on next run by the deferred `os.Remove`) and the original is untouched.

Signature (from `internal/fsutil/atomic_write.go`):

```go
func WriteFileAtomically(path string, content []byte, defaultMode os.FileMode) error
```

The same-directory requirement means the parent directory must already exist. In all three intent sites and the quest site, `os.MkdirAll` is called before the write, so this is already satisfied.

### Step 1: Verify line numbers against the worktree

The review cited `service.go:120,292,354` at baseline `5c22d2b`. The worktree branch `bc2ad1f` has drifted. Verify before editing:

```bash
grep -n "os.WriteFile" \
  projects/worktrees/camp/camp-hardening/internal/intent/service.go \
  projects/worktrees/camp/camp-hardening/internal/quest/quest.go
```

Expected output (at `bc2ad1f`):

```
internal/intent/service.go:90:  if err := os.WriteFile(finalPath, []byte(content), 0644)
internal/intent/service.go:258: if err := os.WriteFile(intent.Path, data, 0644)
internal/intent/service.go:317: if err := os.WriteFile(newPath, data, 0644)
internal/quest/quest.go:271:    if err := os.WriteFile(path, data, 0644)
```

If any line numbers differ, relocate by function name before editing:
- `:90` is inside `CreateDirect`
- `:258` is inside `Save`
- `:317` is inside a rename/move path (look for `os.MkdirAll` immediately above)
- `quest.go:271` is inside `save` (lowercase, the private helper)

### Step 2: Add the fsutil import where missing

Check existing imports in both files:

```bash
grep -n "fsutil" \
  projects/worktrees/camp/camp-hardening/internal/intent/service.go \
  projects/worktrees/camp/camp-hardening/internal/quest/quest.go
```

`service.go` may not yet import `fsutil`. If absent, add:

```go
"github.com/Obedience-Corp/camp/internal/fsutil"
```

`quest.go` already imports several internal packages; add the same line if missing.

### Step 3: Convert the three intent sites

In `internal/intent/service.go`, replace each `os.WriteFile` with `fsutil.WriteFileAtomically`. The mode argument stays `0644`:

Before (at `:90` in `CreateDirect`):
```go
if err := os.WriteFile(finalPath, []byte(content), 0644); err != nil {
    return nil, camperrors.Wrap(err, "writing intent file")
}
```

After:
```go
if err := fsutil.WriteFileAtomically(finalPath, []byte(content), 0644); err != nil {
    return nil, camperrors.Wrap(err, "writing intent file")
}
```

Apply the identical substitution pattern at `:258` in `Save`:
```go
// Before
if err := os.WriteFile(intent.Path, data, 0644); err != nil {
    return camperrors.Wrap(err, "writing intent file")
}

// After
if err := fsutil.WriteFileAtomically(intent.Path, data, 0644); err != nil {
    return camperrors.Wrap(err, "writing intent file")
}
```

And at `:317` in the rename path:
```go
// Before
if err := os.WriteFile(newPath, data, 0644); err != nil {
    return nil, camperrors.Wrap(err, "writing intent file")
}

// After
if err := fsutil.WriteFileAtomically(newPath, data, 0644); err != nil {
    return nil, camperrors.Wrap(err, "writing intent file")
}
```

Note on the `:317` site: the function first derives `newPath`, calls `os.MkdirAll` at `:307`, serializes data, then writes. This is a create-then-remove-old pattern (atomic move semantics). The write is already to a new path before the old file is removed, so the primary crash window here is the write itself. Converting to atomic write adds an extra rename but is still correct. The subsequent `os.Remove(oldPath)` at `:322` is not in scope for this task.

### Step 4: Convert the quest site

In `internal/quest/quest.go`, the `save` helper at `:271`:

```go
// Before
if err := os.WriteFile(path, data, 0644); err != nil {
    return camperrors.Wrapf(err, "write quest file %s", path)
}

// After
if err := fsutil.WriteFileAtomically(path, data, 0644); err != nil {
    return camperrors.Wrapf(err, "write quest file %s", path)
}
```

Add `"github.com/Obedience-Corp/camp/internal/fsutil"` to the imports if not present.

### Step 5: Verify no os import is now unused

Run `go build` in the worktree to confirm the `os` import is still needed (it is, for `os.MkdirAll` and `os.Stat` elsewhere in both files).

### Step 6: Flag read-modify-write sites for sequence-06 locking

`Save` at `:258` is called after a load-then-serialize cycle. The caller (`Update`, `EditWithEditor`, `UpdateDirect`) loads an intent, modifies fields in memory, then calls `Save`. This is a read-modify-write and needs a lock to prevent two concurrent edits from losing one change. It is NOT trivially lockable here (the lock must span the load-modify-save transaction, which is in the caller). Add a comment so task 03/04 of this sequence or a later sequence can pick it up:

```go
// TODO(seq06-lock): this write is part of a read-modify-write cycle (load -> modify -> Save).
// Adding a file lock here alone does not close the race; the lock must span the load and
// the mutation in the caller. Track under sequence-06 locking work.
if err := fsutil.WriteFileAtomically(intent.Path, data, 0644); err != nil {
```

Do the same for the quest `save` helper if it has callers that load then mutate.

### Step 7: Run the both-profile gate

```bash
# From the worktree root
just build
BUILD_TAGS=dev just build
go test -count=1 -short ./internal/intent/... ./internal/quest/...
go test -tags dev -count=1 -short ./internal/intent/... ./internal/quest/...
```

All must pass before moving to tests.

### Step 8: Add focused tests

Test placement follows the repo convention: host `t.TempDir()` for unit tests, `//go:build integration` in `tests/integration/` for container tests. These tests are unit-level.

**Test 1: Content integrity after a successful write**

In `internal/intent/service_test.go` (add to the existing table-driven suite or create a focused test):

```go
func TestCreateDirectAtomic(t *testing.T) {
    svc, dir := newTestService(t)
    ctx := context.Background()

    intent, err := svc.CreateDirect(ctx, CreateOptions{
        Title: "atomic test",
        Type:  TypeTask,
    })
    if err != nil {
        t.Fatalf("CreateDirect: %v", err)
    }

    got, err := os.ReadFile(intent.Path)
    if err != nil {
        t.Fatalf("ReadFile: %v", err)
    }
    if !bytes.Contains(got, []byte("atomic test")) {
        t.Errorf("content missing title: %s", got)
    }
    _ = dir
}
```

**Test 2: Partial-write safety (bad temp-dir injection)**

`fsutil.WriteFileAtomically` creates the temp file in the same directory as the destination (via `os.CreateTemp(dir, ...)`). If we make the parent directory read-only after the intent directory exists, the temp file creation will fail and the original file must remain unchanged. This verifies the "original intact on failure" property:

```go
func TestSaveAtomicFailurePreservesOriginal(t *testing.T) {
    svc, _ := newTestService(t)
    ctx := context.Background()

    intent, err := svc.CreateDirect(ctx, CreateOptions{
        Title: "original",
        Type:  TypeTask,
    })
    if err != nil {
        t.Fatalf("CreateDirect: %v", err)
    }

    original, err := os.ReadFile(intent.Path)
    if err != nil {
        t.Fatalf("ReadFile: %v", err)
    }

    // Make the parent directory read-only so the temp file cannot be created.
    dir := filepath.Dir(intent.Path)
    if err := os.Chmod(dir, 0555); err != nil {
        t.Skipf("chmod not supported: %v", err)
    }
    defer os.Chmod(dir, 0755)

    intent.Title = "mutated"
    saveErr := svc.Save(ctx, intent)
    if saveErr == nil {
        t.Fatal("expected Save to fail with read-only parent")
    }

    got, err := os.ReadFile(intent.Path)
    if err != nil {
        t.Fatalf("ReadFile after failed save: %v", err)
    }
    if !bytes.Equal(original, got) {
        t.Error("original content was corrupted by a failed save")
    }
}
```

Add an analogous test in `internal/quest/` for `save`.

## Edge Cases

- **macOS `/var` symlink**: the parent-directory check in `WriteFileAtomically` uses `os.Stat` not `EvalSymlinks`. This is fine because the temp file is created in the same dir as the destination; the rename is local and does not cross filesystem boundaries.
- **Mode on new files**: for `CreateDirect`, the destination does not yet exist, so `WriteFileAtomically` uses `defaultMode` (0644). Verify that 0644 matches the existing `os.WriteFile` call's mode arg (it does).
- **Concurrent create race**: `CreateDirect` stat-guards at `:85-87` (`os.Stat` + `ErrFileExists`). Converting the write to atomic does not close the stat-then-create TOCTOU (two concurrent creates can both pass the stat guard). That race is tracked under R33/sequence-11; do NOT add an `O_EXCL` open here.

## Out of Scope

- `internal/intent/migration_filesystem.go:91` and `internal/intent/migration.go:389`: these are migration utilities that run during repair, not runtime state writers; they are out of scope for this task.
- `internal/intent/promote/promote.go:230`: promote README write; out of scope (WF-7 territory, sequence 05).
- `internal/intent/feedback/tracker.go:65`, `builder.go:44,56`: feedback subsystem; not in WF-6 scope.
- Locking the read-modify-write cycle in `Save` callers: flagged with TODO comments here; actual locking is not in scope for this task.
- `rename.go` and `notes.go` cited in the first-pass review: these files do not exist at worktree `bc2ad1f`; the equivalent functionality is in `service.go` and `update.go`. Verify with `ls internal/intent/` before assuming the old names apply.

## Done When

- [x] All requirements met.
- [x] `grep -n "os.WriteFile" internal/intent/service.go internal/quest/quest.go` in the worktree returns zero results.
- [x] Both-profile gate passes: `just build`, `go test -count=1 -short ./internal/intent/... ./internal/quest/...`, and `go test -tags dev -count=1 -short ./internal/intent/... ./internal/quest/...` all green.
- [x] New tests for content integrity and partial-write safety exist and pass.
- [x] `TODO(seq06-lock)` comments are present on the `Save` and quest `save` write sites.

## Completion Notes

- Project commit: `8177bb1` (`fix: make intent and quest writes atomic`).
- Converted intent `CreateDirect`, `Save`, and `Move` writes plus quest `writeQuestFile` to `fsutil.WriteFileAtomically(..., 0644)`.
- Added focused tests:
  - `TestIntentService_CreateDirectAtomicWriteContent`
  - `TestIntentService_SaveAtomicFailurePreservesOriginal`
  - `TestSaveAtomicWriteContent`
  - `TestSaveAtomicFailurePreservesOriginal`
- Verification:
  - `rg -n "os\\.WriteFile" internal/intent/service.go internal/quest/quest.go || true`
  - `rg -n "TODO\\(seq06-lock\\)|WriteFileAtomically|fsutil" internal/intent/service.go internal/quest/quest.go`
  - `go test -count=1 -short ./internal/intent/... ./internal/quest/...`
  - `go test -tags dev -count=1 -short ./internal/intent/... ./internal/quest/...`
  - `go test -v -count=1 -run 'Test(IntentService_CreateDirectAtomicWriteContent|IntentService_SaveAtomicFailurePreservesOriginal)$|TestSaveAtomic' ./internal/intent ./internal/quest`
  - `just build stable`
  - `just build dev`
  - `git diff --check`

Note: `just build` itself is a build namespace help target in this worktree; the executable gate targets are `just build stable` and `just build dev`.
