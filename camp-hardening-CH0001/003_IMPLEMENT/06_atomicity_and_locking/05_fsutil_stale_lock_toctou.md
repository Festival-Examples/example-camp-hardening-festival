---
fest_type: task
fest_id: 05_fsutil_stale_lock_toctou.md
fest_name: fsutil_stale_lock_toctou
fest_parent: 06_atomicity_and_locking
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.69612-06:00
fest_tracking: true
---

# Task: Fix fsutil Stale-Lock TOCTOU (REG-2)

## Objective

Replace the stat-then-remove stale-lock removal in `internal/fsutil/lock.go` with a rename-steal approach so that exactly one concurrent caller wins the steal and no fresh lock is accidentally removed; make `staleLockAfter` a named constant with documented rationale.

## Requirements

- [x] **REG-2a**: `removeStaleLock` replaced with a race-free steal: rename the candidate stale lock to a unique path, verify the renamed file is still stale, then claim the lock by creating a fresh `O_CREATE|O_EXCL` lock file. If any step fails, treat the lock as active and return `(false, nil)`.
- [x] **REG-2b**: `staleLockAfter = 30 * time.Second` remains a named constant with an explanatory comment documenting why 30 seconds was chosen and when a legitimate holder would exceed it.
- [x] **REG-2c**: stat errors (permissions, unexpected errors) are treated as "lock is active" (return `false, nil`), maintaining the fail-safe behavior that already exists in the current code.
- [x] Behavior change is minimal: under normal operation (no concurrent stealers, no very-slow lock holders), the observable behavior of `AcquireFileLock` is unchanged. The fix matters only when two callers simultaneously observe a stale lock.
- [x] Existing consumers of `AcquireFileLock` (`links.WithLock`, `doctor_ref_backfill.go`, the new `UpdateRegistry` and `priority.WithLock` from tasks 03-04) require no changes.
- [x] New tests: simulated steal race (two goroutines both trying to steal the same stale lock; exactly one wins); a lock held within the threshold is NOT stolen; context cancellation during acquire returns promptly.

## Implementation

### Background: the TOCTOU race and why it matters

The current `removeStaleLock` in `internal/fsutil/lock.go:71-86`:

```go
func removeStaleLock(lockPath string) (bool, error) {
    info, err := os.Stat(lockPath)
    if errors.Is(err, fs.ErrNotExist) {
        return true, nil
    }
    if err != nil {
        return false, camperrors.Wrap(err, "stat lock")
    }
    if time.Since(info.ModTime()) <= staleLockAfter {
        return false, nil
    }
    if err := os.Remove(lockPath); err != nil && !errors.Is(err, fs.ErrNotExist) {
        return false, camperrors.Wrap(err, "remove stale lock")
    }
    return true, nil
}
```

The race: between `os.Stat` (sees stale lock from process A, which crashed 35 seconds ago) and `os.Remove`, process B can:
1. Remove the stale lock first.
2. Create a fresh `O_CREATE|O_EXCL` lock file (now held by B).
3. This process then `os.Remove`s B's fresh lock.
4. Both this process and some third process C now both try `O_CREATE|O_EXCL`, and both can succeed because the lock file is gone.

Result: two holders of the same lock. For short-lived locks (registry, priority store) the practical impact is rare but not zero, especially in agent loops where many camp processes fire in parallel.

The fix uses `os.Rename` as an atomic "claim" operation. Only one caller's rename can succeed when two try to rename the same source to different temp paths; the others get `ErrNotExist` (the source was already renamed away) and treat the lock as active, backing off.

### Step 1: Read the current lock implementation

Read `internal/fsutil/lock.go` in full to understand the complete flow before making changes. Key points:
- `AcquireFileLock` retries with 20ms sleep up to `defaultLockTimeout` (5s).
- `tryAcquireFileLock` attempts `O_CREATE|O_EXCL`; on `ErrExist`, calls `removeStaleLock`.
- `removeStaleLock` is the only function that needs to change.

```bash
cat projects/worktrees/camp/camp-hardening/internal/fsutil/lock.go
```

### Step 2: Document the staleLockAfter constant

Add a comment above the constant explaining the rationale:

```go
// staleLockAfter is the age beyond which a lock file is considered abandoned.
// 30 seconds is intentionally conservative: camp's longest lock-holding operations
// (registry saves, priority store saves, link saves) complete in well under 1 second.
// Legitimate holders that exceed this threshold are processes that crashed or were
// SIGKILL'd; those locks will never be released voluntarily.
// If a future operation needs to hold a lock longer, raise this value or use a
// separate lock with a custom timeout rather than stretching this global threshold.
staleLockAfter = 30 * time.Second
```

### Step 3: Replace removeStaleLock with rename-steal

Replace the function body with the following logic. The steal works in four steps:
1. Stat the lock file. If it does not exist, another caller already stole it; return `(false, nil)` to let the retry loop try `O_CREATE|O_EXCL` again on the next cycle.
2. Check the age. If within threshold, the lock is active; return `(false, nil)`.
3. Rename the stale lock to a unique temp path (using `os.CreateTemp` for the name, then renaming onto it). Only one caller's rename succeeds; all others get `ErrNotExist` and back off.
4. Verify the renamed file is still stale (guarding against a race where the clock ticked backward or the mtime was updated). If the renamed file is NOT stale, rename it back and return `(false, nil)`.

```go
func removeStaleLock(lockPath string) (bool, error) {
    info, err := os.Stat(lockPath)
    if errors.Is(err, fs.ErrNotExist) {
        // Another stealer already removed it; let the retry loop proceed.
        return false, nil
    }
    if err != nil {
        // Treat permission errors and other stat failures as "lock is active".
        return false, nil
    }
    if time.Since(info.ModTime()) <= staleLockAfter {
        return false, nil
    }

    // Rename-steal: atomically claim the stale lock by renaming it to a unique path.
    // Only one concurrent stealer wins; all others get ErrNotExist here and back off.
    dir := filepath.Dir(lockPath)
    stealPath := filepath.Join(dir, filepath.Base(lockPath)+".steal-"+randomHex(8))
    if err := os.Rename(lockPath, stealPath); err != nil {
        if errors.Is(err, fs.ErrNotExist) {
            // Another stealer won; back off.
            return false, nil
        }
        // Any other error: treat as active lock.
        return false, nil
    }

    // Verify the stolen file is still stale (defensive: mtime should not have changed,
    // but be safe on networked filesystems with coarse time granularity).
    stolenInfo, err := os.Stat(stealPath)
    if err != nil || time.Since(stolenInfo.ModTime()) <= staleLockAfter {
        // Not actually stale or cannot verify; rename back and treat as active.
        _ = os.Rename(stealPath, lockPath)
        return false, nil
    }

    // We own the stolen file; remove it to free the path for a fresh lock.
    _ = os.Remove(stealPath)
    return false, nil
}
```

Return value note: `removeStaleLock` returning `(false, nil)` after a successful steal is intentional. The function signals "did I clean up the lock path?" The retry loop in `tryAcquireFileLock` then immediately re-attempts `O_CREATE|O_EXCL`. Returning `(true, nil)` would cause the caller to skip a retry cycle; returning `(false, nil)` causes an immediate retry, which is correct because the path is now free.

Add the `randomHex` helper (or use `os.CreateTemp` style):

```go
func randomHex(n int) string {
    b := make([]byte, n)
    if _, err := rand.Read(b); err != nil {
        return fmt.Sprintf("%d", time.Now().UnixNano())
    }
    return fmt.Sprintf("%x", b)
}
```

Import `"crypto/rand"` and `"fmt"` (or `math/rand` with a seeded source). The crypto/rand version is preferred; if it fails, falling back to a timestamp is acceptable because the steal is best-effort.

Add `"path/filepath"` to imports if not already present.

### Step 4: Verify no behavior change in normal operation

Under normal operation (no stale locks), `removeStaleLock` is only called when `O_CREATE|O_EXCL` returns `ErrExist`, meaning a lock file already exists. If the lock file is fresh (within 30 seconds), the function returns `(false, nil)` immediately, same as before. If the lock file is older than 30 seconds, the rename-steal runs. The only observable difference from the caller's perspective is that after a successful steal, the caller gets `(false, nil)` and retries `O_CREATE|O_EXCL` immediately, which succeeds (path is now clear). The old code returned `(true, nil)` after removing the stale lock, causing the retry loop to exit `tryAcquireFileLock` with `removed == true` and also retry. The net behavior for a single caller is the same: one retry cycle to claim the lock after a stale removal.

### Step 5: Run the both-profile gate

```bash
just build
BUILD_TAGS=dev just build
go test -count=1 -short ./internal/fsutil/...
go test -tags dev -count=1 -short ./internal/fsutil/...
```

### Step 6: Add focused tests

Test placement: host `t.TempDir()` unit tests. Add to `internal/fsutil/` (create `lock_test.go` if it does not exist).

**Test: a lock held within the threshold is NOT stolen**

```go
func TestAcquireFileLockNotStolenIfFresh(t *testing.T) {
    dir := t.TempDir()
    lockPath := filepath.Join(dir, "test.lock")

    // Create a fresh lock file (well within the 30-second threshold).
    f, err := os.OpenFile(lockPath, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0o644)
    if err != nil {
        t.Fatal(err)
    }
    f.Close()

    // Attempting to acquire should time out (the lock is fresh, not stale).
    ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
    defer cancel()
    _, err = AcquireFileLock(ctx, lockPath)
    if err == nil {
        t.Fatal("expected timeout error acquiring a fresh lock")
    }
    // Verify the lock file still exists (was NOT stolen).
    if _, statErr := os.Stat(lockPath); os.IsNotExist(statErr) {
        t.Error("fresh lock was incorrectly stolen")
    }
}
```

**Test: two goroutines stealing the same stale lock; exactly one gets the subsequent acquire**

```go
func TestStaleLockExactlyOneWinner(t *testing.T) {
    dir := t.TempDir()
    lockPath := filepath.Join(dir, "test.lock")

    // Create a stale lock file (older than staleLockAfter).
    f, err := os.OpenFile(lockPath, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0o644)
    if err != nil {
        t.Fatal(err)
    }
    f.Close()
    staleTime := time.Now().Add(-60 * time.Second)
    if err := os.Chtimes(lockPath, staleTime, staleTime); err != nil {
        t.Skipf("Chtimes not supported: %v", err)
    }

    ctx := context.Background()
    var wg sync.WaitGroup
    results := make([]error, 2)
    releases := make([]func(), 2)
    wg.Add(2)
    for i := 0; i < 2; i++ {
        i := i
        go func() {
            defer wg.Done()
            rel, err := AcquireFileLock(ctx, lockPath)
            results[i] = err
            releases[i] = rel
        }()
    }
    wg.Wait()

    successCount := 0
    for i, err := range results {
        if err == nil {
            successCount++
            if releases[i] != nil {
                releases[i]()
            }
        }
    }
    if successCount != 2 {
        // Both should eventually succeed (one wins steal, the other retries and gets it after release).
        // If exactly 0 or 1 succeed within the timeout, that is a bug.
        t.Errorf("expected both goroutines to acquire the lock (serially), got %d successes", successCount)
    }
}
```

Note: the test does not assert that exactly one goroutine wins the steal simultaneously; it asserts that both goroutines eventually succeed (the steal winner takes it, releases, and the loser acquires on the next retry cycle). Both succeeding serially is the correct behavior.

**Test: context cancellation during acquire returns promptly**

```go
func TestAcquireFileLockContextCancellation(t *testing.T) {
    dir := t.TempDir()
    lockPath := filepath.Join(dir, "test.lock")

    // Hold the lock in the background.
    rel, err := AcquireFileLock(context.Background(), lockPath)
    if err != nil {
        t.Fatal(err)
    }
    defer rel()

    ctx, cancel := context.WithCancel(context.Background())
    cancel() // cancel immediately

    _, err = AcquireFileLock(ctx, lockPath)
    if err == nil {
        t.Fatal("expected error on cancelled context")
    }
}
```

## Edge Cases

- **Networked filesystems (NFS, SMB)**: `os.Rename` is not guaranteed to be atomic across NFS mounts. Camp is designed for local filesystems; the comment in `lock.go` should note "NFS-style remote mounts are not in scope; use advisory locking at the OS level for distributed scenarios."
- **macOS `/var` vs `/private/var`**: the steal path is in the same directory as the lock path; no cross-mount rename occurs. The macOS symlink is not relevant here.
- **`stealPath` naming**: using a random hex suffix (not just a fixed `.steal` suffix) ensures two concurrent stealers do not try to rename to the same path, which would cause one rename to fail with `ENOENT` (source already renamed) rather than `EEXIST` (dest already exists). Both outcomes cause correct back-off, but the random suffix makes the intent clearer.
- **Long-running operations**: if any future operation legitimately holds a lock for more than 30 seconds (e.g., a large registry operation over a slow filesystem), it will have its lock stolen. The staleLockAfter constant comment documents this explicitly. Future work: support `WithTimeout` option in `AcquireFileLock` for callers that need custom thresholds.

## Out of Scope

- Supporting distributed or networked filesystem lock semantics: not in scope.
- PID-liveness verification as an alternative to rename-steal: both approaches are valid; rename-steal is chosen because it does not require reading/writing a PID into the lock file (changing the lock file format would break `AcquireFileLock`'s current `O_CREATE|O_EXCL` semantics).
- `internal/git/lock_unix.go` stale-lock removal (GIT-9): that is a different lock mechanism (fuser/lsof based) in the git subsystem. It is not in scope for this task and must not be touched here.
- Changing `defaultLockTimeout` (5 seconds): the timeout is not part of the TOCTOU finding and should not change.

## Completion Notes

- Project commits:
  - `975f49f` (`fix: steal stale file locks safely`)
  - `19d085f` (`test: cover serial stale lock acquisition`)
- Replaced stat-then-remove stale cleanup with a same-directory rename-steal using unique `.steal-<random>` names.
- After a successful steal, `tryAcquireFileLock` immediately claims the public lock path with the existing `O_CREATE|O_EXCL` mechanism.
- `removeStaleLock` treats stat, rename, verification, and cleanup failures as an active lock by returning `(false, nil)`.
- Added the documented `staleLockAfter` rationale and a local-filesystem note for the rename semantics.
- Added tests for fresh-lock protection, exactly-one stale stealer, serial acquisition through `AcquireFileLock`, and cancellation while waiting.

## Verification

- `go test -count=1 -short ./internal/fsutil/...`
- `go test -tags dev -count=1 -short ./internal/fsutil/...`
- `go test -v -count=1 -run 'TestAcquireFileLock|TestTryAcquireFileLock' ./internal/fsutil`
- `go test -v -count=1 -run 'TestAcquireFileLock_StaleLockStealRace|TestTryAcquireFileLock_StaleLockStealRace' ./internal/fsutil`
- `just build stable`
- `just build dev`
- `git diff --check`

## Done When

- [x] All requirements met.
- [x] `removeStaleLock` uses rename-steal; the old stat-then-remove code is gone.
- [x] `staleLockAfter` constant has an explanatory comment.
- [x] Both-profile gate passes for `internal/fsutil/...`.
- [x] Test: fresh lock not stolen.
- [x] Test: stale lock steal race - both goroutines acquire serially.
- [x] Test: context cancellation returns promptly.
- [x] No changes to `links.WithLock`, `UpdateRegistry`, `priority.WithLock`, or any consumer of `AcquireFileLock` (the API surface is unchanged).
