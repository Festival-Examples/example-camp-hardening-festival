---
fest_type: task
fest_id: 01_only_index_snapshot.md
fest_name: only_index_snapshot
fest_parent: 07_commit_semantics
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.713172-06:00
fest_tracking: true
---

# Task: Replace --only with temp-index snapshot engine (N-4, D001 Option A)

## Objective

Replace `git commit --only` in the scoped-commit path with a temporary `GIT_INDEX_FILE` snapshot so that auto-commit flows capture the add-time state and `--staged` commits exactly the staged blobs, closing the worktree-content over-commit bug (N-4).

## Requirements

- [x] `stageAndCommit` in `internal/git/commit/commit.go` no longer passes `Only` to `git.Commit`; instead it builds a temp index, populates it, and commits from it.
- [x] Auto-commit flow (when `opts.Files` is non-empty): temp index is populated via `GIT_INDEX_FILE=<tmp> git read-tree HEAD` then `git add -- <paths>`, so the commit captures only the worktree content at add time.
- [x] `--staged` flow (when `opts.PreStaged` is non-empty and `opts.Files` is empty): temp index is a copy of the real index (located via `ResolveGitDir`), so exactly the staged blobs commit; a cheap divergence assertion fires if no staged content is found in the copy.
- [x] `git.CommitOptions.Only` field is removed or left for back-compat but the `--only --` argv in `executeCommit` is replaced by a `TempIndexPath` env override field.
- [x] `git.Commit` and the `WithLockRetry` closure propagate a `GIT_INDEX_FILE` env var when one is provided.
- [x] Temp file is always cleaned up (deferred `os.Remove`) even on retry failure or context cancellation.
- [x] `ResolveGitDir` (at `internal/git/lock.go:89`) is the authoritative way to locate the actual `.git/index`; never hardcode `.git/index` in new code.
- [x] Commit message formatting (campaign tag, quest, festival ref, workitem ref) is unchanged; only the commit mechanics change.
- [x] Regression tests cover: staged-v2/worktree-v3 `--staged` commits v2; auto-commit captures add-time snapshot (not later worktree state); a concurrent-writer window test where feasible.

## Implementation

### Background: why `--only` does not have the semantics the code assumes

`git commit --only -- <paths>` does NOT commit the staged (index) content of those paths. It commits the CURRENT WORKTREE content. Empirically reproduced (second-pass review, scratch repo):

```
echo v2 > f.txt; git add f.txt          # f.txt staged at v2
echo v3 > f.txt                          # worktree now at v3 (not staged)
git commit --only -- f.txt -m "test"     # commits v3, silently restages v3
```

Every call to `git.Commit` with a non-empty `Only` field today hits this path. The internal docs and the `stageAndCommit` comment say "index semantics" but the actual behavior is worktree semantics. D001 Option A is the accepted fix.

### Verified file:line targets (worktree HEAD bc2ad1f)

- `internal/git/commit/commit.go:83-107` -- `stageAndCommit` function; `commitScope` built at :84, `git.StageFiles` at :87, `ExpandTrackedPaths` at :91-94, `git.Commit` with `Only:` at :98-101, `SelectiveOnly` path at :103-105, `CommitAll` fallback at :106.
- `internal/git/commit.go:30-44` -- `Commit` function with `WithLockRetry`; retry closure at :41.
- `internal/git/commit.go:47-87` -- `executeCommit`; the `--only --` argv is built at :62-65.
- `internal/git/commit.go:12-19` -- `CommitOptions` struct; `Only []string` field at :18.
- `internal/git/lock.go:89-109` -- `ResolveGitDir`; reads `.git` file for submodules at :99-108.

Note: the review cited `internal/git/commit/commit.go:83-107` (first-pass) and `internal/git/commit.go:62-65`. Both match the worktree exactly at bc2ad1f. No drift.

### Step 1: Add TempIndexPath to CommitOptions and update executeCommit

In `internal/git/commit.go`, replace the `Only []string` handling in `executeCommit` with a `TempIndexPath string` field on `CommitOptions`:

```go
// CommitOptions configures the commit operation.
type CommitOptions struct {
    Message       string
    Amend         bool
    AllowEmpty    bool
    Author        string
    TempIndexPath string // When set, passes GIT_INDEX_FILE=<path> to git commit.
                        // The caller owns temp file lifecycle; this field is never
                        // set by callers directly -- use the commit engine helpers.
    // Only is retained for external callers but is no longer used internally.
    // Deprecated: use the temp-index engine via stageAndCommit instead.
    Only []string
}
```

In `executeCommit`, replace the `--only --` block:

```go
// Before (remove this):
if len(opts.Only) > 0 {
    args = append(args, "--only", "--")
    args = append(args, opts.Only...)
}

// After: propagate temp index via env, not argv.
cmd := exec.CommandContext(ctx, "git", args...)
if opts.TempIndexPath != "" {
    cmd.Env = append(os.Environ(), "GIT_INDEX_FILE="+opts.TempIndexPath)
}
output, err := cmd.CombinedOutput()
```

`os.Environ()` ensures the subprocess inherits all existing env vars; only `GIT_INDEX_FILE` is overridden.

### Step 2: Add a helper to build temp index paths

Add `BuildTempIndexPath` in `internal/git/commit.go` (or a new `internal/git/tempindex.go`):

```go
// BuildTempIndexPath resolves the real git index path for repoPath and returns
// a sibling temp-file path suitable for use as GIT_INDEX_FILE. The caller must
// create and populate the file; call RemoveTempIndex when done.
func BuildTempIndexPath(repoPath string) (tempPath string, realIndexPath string, err error) {
    gitDir, err := ResolveGitDir(repoPath)
    if err != nil {
        return "", "", err
    }
    realIndex := filepath.Join(gitDir, "index")
    f, err := os.CreateTemp(gitDir, "index.tmp.*")
    if err != nil {
        return "", "", camperrors.Wrapf(err, "create temp index in %s", gitDir)
    }
    f.Close()
    return f.Name(), realIndex, nil
}

// RemoveTempIndex removes the temp index file, ignoring not-found errors.
func RemoveTempIndex(path string) {
    _ = os.Remove(path)
}
```

Placing the temp file inside `gitDir` ensures it is on the same filesystem as the real index, which matters for git's internal operations.

### Step 3: Rewrite stageAndCommit in internal/git/commit/commit.go

The function currently at `commit.go:83-107` must handle two modes:

**Mode A (auto-commit, `opts.Files` non-empty):** build a fresh temp index from HEAD, add the requested paths into it, commit from that index.

**Mode B (--staged, `opts.PreStaged` non-empty, `opts.Files` empty):** copy the real index into the temp index, commit from the copy. Add a cheap assertion that the copy is not empty.

```go
func stageAndCommit(ctx context.Context, opts Options, message string) error {
    commitScope := append(append([]string{}, opts.Files...), opts.PreStaged...)
    if len(commitScope) == 0 {
        if opts.SelectiveOnly {
            return git.ErrNoChanges
        }
        return git.CommitAll(ctx, opts.CampaignRoot, message)
    }

    tmpPath, realIndex, err := git.BuildTempIndexPath(opts.CampaignRoot)
    if err != nil {
        return err
    }
    defer git.RemoveTempIndex(tmpPath)

    if len(opts.Files) > 0 {
        // Mode A: snapshot worktree content of Files at add-time.
        // read-tree HEAD into temp index, then add only the requested paths.
        if err := git.ReadTreeIntoTempIndex(ctx, opts.CampaignRoot, tmpPath); err != nil {
            return err
        }
        if err := git.AddPathsToTempIndex(ctx, opts.CampaignRoot, tmpPath, opts.Files); err != nil {
            return err
        }
    } else {
        // Mode B (--staged): commit exactly the current staged blobs.
        // Copy real index to temp; assert that at least one of commitScope is staged.
        if err := copyFile(realIndex, tmpPath); err != nil {
            return camperrors.Wrapf(err, "copy real index to temp")
        }
        // Divergence assertion: confirm something is staged for commitScope.
        expanded, err := git.ExpandTrackedPathsFromTempIndex(ctx, opts.CampaignRoot, tmpPath, commitScope)
        if err != nil {
            return err
        }
        if len(expanded) == 0 {
            return git.ErrNoChanges
        }
    }

    return git.Commit(ctx, opts.CampaignRoot, &git.CommitOptions{
        Message:       message,
        TempIndexPath: tmpPath,
    })
}
```

`copyFile` is a helper that copies a file byte-for-byte using `io.Copy` with proper close/error handling. Do not use `os.Rename` because the real index must remain in place.

### Step 4: Add git helpers for temp-index operations

Add these to `internal/git/commit.go` (or `tempindex.go`):

```go
// ReadTreeIntoTempIndex runs: GIT_INDEX_FILE=tmpPath git read-tree HEAD
// This seeds the temp index with the committed tree so subsequent adds
// are delta-only against HEAD.
func ReadTreeIntoTempIndex(ctx context.Context, repoPath, tmpPath string) error {
    cmd := exec.CommandContext(ctx, "git", "-C", repoPath, "read-tree", "HEAD")
    cmd.Env = append(os.Environ(), "GIT_INDEX_FILE="+tmpPath)
    out, err := cmd.CombinedOutput()
    if err != nil {
        return camperrors.NewGit("read-tree", "", "", strings.TrimSpace(string(out)), err)
    }
    return nil
}

// AddPathsToTempIndex runs: GIT_INDEX_FILE=tmpPath git add -- <paths>
// This captures the worktree content of paths at call time.
func AddPathsToTempIndex(ctx context.Context, repoPath, tmpPath string, paths []string) error {
    cfg := DefaultRetryConfig()
    cfg.OperationName = "add-temp-index"
    return WithLockRetry(ctx, repoPath, cfg, func() error {
        args := []string{"-C", repoPath, "add", "--", "--"}
        // The first "--" is the -C sentinel; the second guards paths.
        args = []string{"-C", repoPath, "add", "--"}
        args = append(args, paths...)
        cmd := exec.CommandContext(ctx, "git", args...)
        cmd.Env = append(os.Environ(), "GIT_INDEX_FILE="+tmpPath)
        out, err := cmd.CombinedOutput()
        if err != nil {
            errType := ClassifyGitError(string(out), cmd.ProcessState.ExitCode())
            if errType == GitErrorLock {
                return &LockError{Path: "index.lock", Err: err}
            }
            return camperrors.NewGit("add (temp index)", "", errType.String(), strings.TrimSpace(string(out)), err)
        }
        return nil
    })
}

// ExpandTrackedPathsFromTempIndex runs diff --cached against the given temp
// index and returns the expanded scope. Reuses the same -z parser as
// ExpandTrackedPaths so rename pairs are included correctly.
func ExpandTrackedPathsFromTempIndex(ctx context.Context, repoPath, tmpPath string, paths []string) ([]string, error) {
    if len(paths) == 0 {
        return nil, nil
    }
    args := []string{"-C", repoPath, "diff", "--cached", "--name-status", "-z", "--"}
    args = append(args, paths...)
    cmd := exec.CommandContext(ctx, "git", args...)
    cmd.Env = append(os.Environ(), "GIT_INDEX_FILE="+tmpPath)
    output, err := cmd.Output()
    if err != nil {
        return nil, camperrors.NewGit("diff --cached (temp index)", "", "", strings.TrimSpace(string(output)), err)
    }
    return parseNameStatusZ(output)
}
```

`parseNameStatusZ` is the same -z parser logic extracted from `ExpandTrackedPaths`. Extract it into a shared unexported function to avoid duplication.

### Step 5: Lock-retry and env propagation

`WithLockRetry` in `internal/git/retry.go` wraps a closure; the closure captures `tmpPath` by value, so each retry attempt sees the same temp-index path. The temp file must NOT be cleaned up between retries; only clean up after the final result (success or max-retry error). The `defer git.RemoveTempIndex(tmpPath)` in `stageAndCommit` ensures this regardless of retry count.

Verify `WithLockRetry`'s signature accepts a simple `func() error` and that `executeCommit` is what goes inside the closure. The `cmd.Env` assignment inside `executeCommit` fires on each retry, which is correct.

### Step 6: Remove the --only argv from executeCommit

After adding `TempIndexPath` support, remove or stub the `Only` block from `executeCommit`. If removing entirely, audit callers of `git.Commit` that pass `Only:` and update them to use the new engine helpers. The review's verified-clean list confirms `ExpandTrackedPaths` is correct in isolation and does not need rewriting.

### Edge cases

**Worktrees (not submodules):** `ResolveGitDir` handles both. A worktree's `.git` is a file pointing into `.git/worktrees/<name>`; `BuildTempIndexPath` will create the temp file in that directory, which is the same filesystem as the worktree's real index. Correct.

**Empty scope after add:** If `git add --` produces nothing staged in the temp index (all paths are untracked with no changes), `ExpandTrackedPathsFromTempIndex` returns nil, and `stageAndCommit` returns `ErrNoChanges`. This is the correct behavior for auto-commit on unchanged files.

**Concurrent writers during the add window:** Mode A closes the concurrent-writer window relative to `--only`: the add into the temp index captures whatever the file contains at that moment, and the subsequent commit reads from the temp index (not the live worktree). A concurrent process that writes the file after `AddPathsToTempIndex` completes cannot affect this commit. This is the key safety property D001 Option A provides.

**Hook interactions:** git commit hooks receive `GIT_INDEX_FILE` via the child process's environment; pre-commit and post-commit hooks that read the index (e.g., `git diff --cached`) will see the temp index content, which is correct because that IS what is being committed.

**`copyFile` on macOS:** macOS `/var` -> `/private/var` does not affect intra-gitdir copies. Both `realIndex` and `tmpPath` are under `gitDir`, so same device, no EXDEV.

### Regression tests

Add tests in `internal/git/commit/commit_test.go` (host `t.TempDir()`, consistent with the repo's unit-test convention, C-9):

**Test 1: --staged commits staged blobs, not worktree content (N-4 regression)**

```
scratch repo:
  echo v1 > f.txt; git add f.txt; git commit -m "init"
  echo v2 > f.txt; git add f.txt        # f.txt staged at v2
  echo v3 > f.txt                        # worktree now at v3

call stageAndCommit with opts.PreStaged = ["f.txt"], opts.Files = nil
assert: HEAD:f.txt == "v2\n"
assert: worktree f.txt == "v3\n"  (unchanged; commit did not modify worktree)
assert: git diff HEAD -- f.txt shows v3 (v3 stays in worktree, v2 was committed)
```

**Test 2: auto-commit captures add-time snapshot**

```
scratch repo:
  echo v1 > f.txt; git add f.txt; git commit -m "init"
  echo v2 > f.txt                        # worktree at v2, not staged

call stageAndCommit with opts.Files = ["f.txt"]
write "v3" to f.txt concurrently (before the commit fires, if feasible in tests)
assert: HEAD:f.txt == "v2\n"  (captured at add time, not at commit time)
```

For the concurrent-writer case, at minimum test the sequential version: v2 is in worktree at add time; the temp index is populated; a subsequent write to v3 does not affect the committed content.

**Test 3: staged git mv does not leave a stray staged D (integrates with task 02)**

This test lives in task 02 but should run against the new engine. Verify it here as well:

```
scratch repo:
  echo "content" > a.txt; git add a.txt; git commit -m "init"
  git mv a.txt b.txt                     # staged rename

call stageAndCommit with opts.PreStaged = ["a.txt", "b.txt"], opts.Files = nil
assert: b.txt exists in HEAD; a.txt is absent from HEAD
assert: git diff --cached is empty (no stray staged D)
```

## Done When

- [x] All requirements met
- [x] `just build` and `BUILD_TAGS=dev just build` pass
- [x] `just test unit` (or `go test ./internal/git/...`) includes Test 1 and Test 2 and both pass
- [x] `just lint` (stable and dev profile) passes without new warnings
- [x] No `--only --` argv remains in `executeCommit` for the scoped-commit path
- [x] `ResolveGitDir` is used for all temp-index path construction; no hardcoded `.git/index`
- [x] Temp files are always cleaned up: verify with a test that injects an error after `BuildTempIndexPath` and checks `tmpPath` no longer exists
- [x] `commitkit` package still compiles without change (its `Commit` and `SyncSubmoduleRef` call `git.Commit` directly; as long as `CommitOptions` signature is compatible, no change needed there yet -- that is task 03's job)

## Completion Notes

- Project commit: `b7ed55b` (`fix: commit scoped changes from temp index`).
- `stageAndCommit` now commits scoped changes through a temp index and passes `TempIndexPath` to `git.Commit`; `executeCommit` no longer emits `git commit --only --`.
- `BuildTempIndexPath` uses `ResolveGitDir`; no new hardcoded `.git/index` path was introduced.
- Auto-commit mode seeds the temp index from `HEAD`, adds `opts.Files`, and overlays only requested `opts.PreStaged` diffs from the real index so combined scopes keep their previous behavior without sweeping unrelated staged content.
- After a successful temp-index commit, the real index is reset to `HEAD` for the committed paths so committed scoped paths do not remain staged or appear as reverse diffs.
- Regression tests added for staged-v2/worktree-v3 behavior, add-time snapshot behavior with a pre-commit hook mutating the worktree, staged rename cleanup, and temp-index cleanup on `ErrNoChanges`.
- Verification passed: `go test -count=1 -short ./internal/git/... ./internal/git/commit ./pkg/commitkit`, `go test -tags dev -count=1 -short ./internal/git/... ./internal/git/commit`, `just lint`, `BUILD_TAGS=dev just lint`, `just build stable`, and `just build dev`.

## Out of Scope

- fest repo: no edits to `projects/fest/` in this task or any task in this festival (C-5).
- `ExpandTrackedPaths` itself: verified clean in isolation (C-3); do not rewrite it.
- commitkit message injection: verified argv-safe (C-3); leave it unchanged.
- `--only` removal from the `CommitOptions` struct: keep the field for back-compat with any external callers; just stop using it in `executeCommit` for the internal engine path.
