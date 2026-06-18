---
fest_type: task
fest_id: 01_sync_fallback_quarantine.md
fest_name: sync_fallback_quarantine
fest_parent: 03_sync_and_migration_safety
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.637321-06:00
fest_updated: 2026-06-13T12:44:11.574704-06:00
fest_tracking: true
---


# Task: Fix sync stale-ref fallback: quarantine instead of RemoveAll, real submodule machinery, relative URL resolution

**Finding:** N-6

## Objective

Replace `InitFromDefaultBranch`'s unconditional `os.RemoveAll` and bare `git clone` with a quarantine-rename plus real submodule fetch/update so a stale-ref fallback during `camp sync --force` cannot destroy uncommitted content or leave the repo structurally broken.

## Requirements

- [ ] `InitFromDefaultBranch` refuses to RemoveAll a non-empty directory; instead it quarantine-renames the directory to a timestamped sibling (format: `<name>.sync-quarantine-YYYYMMDD-HHMMSS`) before any destructive action.
- [ ] The fallback performs recovery through real git submodule machinery: `git -C <repoDir> fetch <url>` into the existing module gitdir, then `git -C <repoDir> submodule update --init <subPath>`, preserving the `.git/modules/<subPath>` wiring.
- [ ] Relative `.gitmodules` URLs (those starting with `../` or `./`) are resolved against the superproject's remote origin URL, not the process cwd.
- [ ] Per-submodule result reports what actually happened (quarantine occurred, fallback attempted, fallback succeeded, fallback failed with the quarantine dir available for inspection).
- [ ] The quarantine dir path is included in any error message so the user can inspect or restore it.
- [ ] Container test: hostile-remote fixture where upstream has been force-pushed so the recorded SHA is unreachable; a dirty submodule dir (with an uncommitted file) survives via quarantine; the repo is still a real submodule (`.git` is a file, not a directory) after the operation.

## Implementation

### Background

`InitFromDefaultBranch` at `internal/git/submodule_init.go:143-173` is reached from `InitSubmoduleGraceful` (`:107-138`) when `git submodule update` fails with a stale-ref message. The current implementation (worktree at bc2ad1f, verified):

```
// line 161
if err := os.RemoveAll(subDir); err != nil {
    return camperrors.WrapJoinf(ErrSubmoduleRemove, err, "%s", subPath)
}
// line 166
cmd := exec.CommandContext(ctx, "git", "clone", url, subDir)
```

Two problems:
1. `os.RemoveAll(subDir)` runs unconditionally; no emptiness or dirtiness check.
2. `git clone url subDir` with no `cmd.Dir` creates a standalone `.git` DIRECTORY at `subDir`, orphaning `.git/modules/<subPath>` and checking out the remote default branch instead of the recorded SHA. The subPath is no longer a submodule after this; `git submodule status` will not recognize it, and `camp sync --force` on the same repo again will re-enter this path endlessly.

A relative `.gitmodules` URL like `../foo.git` is passed to `getSubmoduleURL` (`:197-220`) which reads `.gitmodules` verbatim. When this URL is passed to `git clone`, git resolves it relative to the process working directory (wherever `camp` was launched), not the superproject remote. This produces a wrong or missing clone target.

The function `CheckoutDefaultBranch` at `:177` is unused in the fallback path and should remain that way; the fallback goal is to land at the RECORDED SHA, not a branch tip.

### Step 1: Add an emptiness check helper

Add a small helper near the top of `internal/git/submodule_init.go`:

```go
// isDirEmpty reports whether dir exists and contains no files or directories
// (ignoring the case where dir does not exist at all).
func isDirEmpty(dir string) (bool, error) {
    entries, err := os.ReadDir(dir)
    if os.IsNotExist(err) {
        return true, nil
    }
    if err != nil {
        return false, err
    }
    return len(entries) == 0, nil
}
```

### Step 2: Add a relative-URL resolver

```go
// resolveSubmoduleURL resolves a potentially relative submodule URL against
// the superproject's remote origin URL. Relative means starting with "../" or "./".
// If the URL is not relative, it is returned unchanged.
func resolveSubmoduleURL(ctx context.Context, repoDir, url string) string {
    if !strings.HasPrefix(url, "../") && !strings.HasPrefix(url, "./") {
        return url
    }
    remoteURL, err := RemoteOriginURL(ctx, repoDir)
    if err != nil || remoteURL == "" {
        return url
    }
    // filepath.Join is wrong here (would strip the scheme); use path.Join on
    // the path component only, then reassemble.
    base := strings.TrimSuffix(remoteURL, "/")
    // Strip trailing component to get the "parent" remote dir.
    slash := strings.LastIndex(base, "/")
    if slash < 0 {
        return url
    }
    parent := base[:slash]
    rel := strings.TrimPrefix(url, "../")
    return parent + "/" + rel
}
```

`RemoteOriginURL` already exists in the git package (used in `reverseSyncLocalURLs`, `internal/sync/sync.go:351`).

### Step 3: Rewrite `InitFromDefaultBranch`

Replace the body of `InitFromDefaultBranch` (`:143-173`) with:

```go
func InitFromDefaultBranch(ctx context.Context, repoDir, subPath string) error {
    if ctx.Err() != nil {
        return ctx.Err()
    }

    if err := pathutil.ValidateSubmodulePath(repoDir, subPath); err != nil {
        return camperrors.WrapJoinf(ErrSubmoduleClone, err, "%s", subPath)
    }

    url, err := getSubmoduleURL(ctx, repoDir, subPath)
    if err != nil {
        return camperrors.WrapJoinf(ErrSubmoduleURL, err, "%s", subPath)
    }
    url = resolveSubmoduleURL(ctx, repoDir, url)

    subDir := filepath.Join(repoDir, subPath)

    empty, err := isDirEmpty(subDir)
    if err != nil {
        return camperrors.WrapJoinf(ErrSubmoduleRemove, err, "checking %s", subPath)
    }

    if !empty {
        // Quarantine-rename instead of RemoveAll so uncommitted content survives.
        ts := time.Now().UTC().Format("20060102-150405")
        quarantine := subDir + ".sync-quarantine-" + ts
        if renErr := os.Rename(subDir, quarantine); renErr != nil {
            return camperrors.WrapJoinf(ErrSubmoduleRemove, renErr,
                "%s: non-empty dir cannot be quarantined (inspect %s)", subPath, quarantine)
        }
    } else {
        if err := os.RemoveAll(subDir); err != nil {
            return camperrors.WrapJoinf(ErrSubmoduleRemove, err, "%s", subPath)
        }
    }

    // Fetch into the existing .git/modules wiring, then update to initialize.
    // This preserves the submodule structure rather than creating a standalone clone.
    fetchCmd := exec.CommandContext(ctx, "git", "-C", repoDir,
        "fetch", "--recurse-submodules=no", url,
        "refs/heads/*:refs/remotes/origin/*")
    if output, fetchErr := fetchCmd.CombinedOutput(); fetchErr != nil {
        // Fetch failed; fall back to re-init and update.
        initCmd := exec.CommandContext(ctx, "git", "-C", repoDir, "submodule", "init", subPath)
        _ = initCmd.Run()
        updateCmd := exec.CommandContext(ctx, "git", "-C", repoDir,
            "submodule", "update", "--init", subPath)
        if out2, updateErr := updateCmd.CombinedOutput(); updateErr != nil {
            return camperrors.WrapJoinf(ErrSubmoduleClone, updateErr,
                "%s: fetch failed (%s) and update failed: %s",
                subPath, strings.TrimSpace(string(output)), strings.TrimSpace(string(out2)))
        }
        return nil
    }

    updateCmd := exec.CommandContext(ctx, "git", "-C", repoDir,
        "submodule", "update", "--init", subPath)
    if output, updateErr := updateCmd.CombinedOutput(); updateErr != nil {
        return camperrors.WrapJoinf(ErrSubmoduleUpdate, updateErr,
            "%s: %s", subPath, strings.TrimSpace(string(output)))
    }

    return nil
}
```

Add `"time"` to the import block.

### Step 4: Container test

Create or extend `tests/integration/sync_fallback_test.go` with a `//go:build integration` tag.

The test scenario:
1. Create a bare origin repo with an initial commit at SHA-A.
2. Add it as a submodule to a superproject; record SHA-A in `.gitmodules`.
3. Force-push the origin to SHA-B (orphaning SHA-A).
4. Populate the submodule working directory with an uncommitted file (`dirty.txt`).
5. Call `InitSubmoduleGraceful` on the submodule path.
6. Assert: the quarantine dir (`<name>.sync-quarantine-*`) exists and contains `dirty.txt`.
7. Assert: the submodule dir exists and `.git` is a FILE (not a directory), confirming it is a real submodule.
8. Assert: `git -C <superproject> submodule status <subPath>` exits 0 (submodule recognized).

The fixture also serves task 03 which needs a repo with state that fails a pull.

### Edge cases

- **Empty subDir**: `isDirEmpty` returns true and the current `os.RemoveAll` path is taken (no behavior change for the common case).
- **Quarantine rename fails**: Return the error immediately; do not attempt the update. The user's content is still in the original location.
- **macOS `/private/var`**: `subDir` is constructed with `filepath.Join(repoDir, subPath)` where `repoDir` comes from camp's root detection which already uses `EvalSymlinks` in the project-resolve path. If a caller passes a non-resolved path, the quarantine rename will still succeed since it is a sibling rename and both sides resolve to the same filesystem.
- **Relative URL with no remote origin**: `resolveSubmoduleURL` returns the original URL unchanged; git will fail the fetch with a clear "not a valid URL" error, which is the correct behavior (do not silently use the wrong path).

## Out of Scope

- Building the shared statusdir move primitive (that is 11.05 per D003); this task's quarantine helper is intentionally narrow.
- Fixing `camp sync` exit code table (sequence 10).
- Fixing the orphan-gitlink cleanup path, which the second-pass verified clean (C-3).

## Done When

- [ ] `isDirEmpty`, `resolveSubmoduleURL`, and the rewritten `InitFromDefaultBranch` land in `internal/git/submodule_init.go`
- [ ] Container test passes under `just test integration`: dirty submodule dir survives quarantine, repo is a real submodule afterward
- [ ] Both-profile gate passes: `just build`, `go test -count=1 -short ./...` (stable), `go test -tags dev -count=1 -short ./...` (dev), `go vet ./...` both profiles
- [ ] No new host-git test files added