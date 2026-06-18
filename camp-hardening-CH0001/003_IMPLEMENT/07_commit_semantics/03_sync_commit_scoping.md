---
fest_type: task
fest_id: 03_sync_commit_scoping.md
fest_name: sync_commit_scoping
fest_parent: 07_commit_semantics
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.714168-06:00
fest_updated: 2026-06-14T23:31:44.266602-06:00
fest_tracking: true
---


# Task: Scope syncParentRef, SyncSubmoduleRef, and refs-sync commits (R6, GIT-2, GIT-7)

## Objective

Make every submodule-ref sync commit in camp scope its write to the submodule gitlink path so that unrelated staged content at the campaign root is never swept into a sync commit, and make refs-sync print which submodules it skipped and why.

## Requirements

- [ ] `syncParentRef` in `cmd/camp/project/commit.go:190-216` commits scoped to `relPath` using the new engine from task 01; the no-op guard checks only whether the ref path has staged content, not whether the whole repo has staged changes.
- [ ] `SyncSubmoduleRef` in `pkg/commitkit/commitkit.go:186-213` commits scoped to `projectRelPath` using the new engine; the `HasStagedChanges` no-op guard at :197 is replaced with a path-scoped check for the ref path only; the unscoped `git.Commit` at :208 is replaced with a scoped commit.
- [ ] `runRefsSync` in `cmd/camp/refs/commands.go` commits with `Only: toSync` (or equivalent scoped engine call); skipped submodules (those where `detectRefChanges` returned `Changed: false` or where errors caused a `continue`) are printed with a reason line rather than silently omitted.
- [ ] `pkg/commitkit` package-level doc comment is updated to document that `SyncSubmoduleRef` now commits only the gitlink path; callers relying on the old whole-repo sweep must be updated.
- [ ] `commitkit.SyncSubmoduleRef` signature is unchanged (same parameters and return type); only the internal semantics change.
- [ ] Container/host test: a pre-staged unrelated file at the campaign root survives a `camp p commit --sync` call and a `SyncSubmoduleRef` call without being committed or unstaged.

## Implementation

### Background: why gitlink scoping is safe

A submodule gitlink is a special tree entry of type `160000` (commit object). It has no relationship to any file path under the submodule. When `git add projects/fest` is run in the campaign root, git records the current HEAD of the `projects/fest` submodule as the gitlink SHA. The worktree content of `projects/fest/` is not affected. Therefore `--only -- projects/fest` (or the temp-index equivalent: `git add -- projects/fest` into a fresh temp index) captures exactly the gitlink and nothing else. This is the one case where the temp-index engine's Mode A (auto-commit) is correct for a sync: the "file" being committed is the gitlink, which does not have a distinct staged-vs-worktree state -- its value IS the submodule HEAD.

The review confirmed this in second-pass correction 3: "for submodule gitlinks this is acceptable (worktree gitlink state is the submodule HEAD)."

### Verified file:line targets (worktree HEAD bc2ad1f)

- `cmd/camp/project/commit.go:190-216` -- `syncParentRef`; `parentExec.Stage(ctx, []string{relPath})` at :196; unscoped `parentExec.Commit(ctx, opts)` at :207. No drift from review citation.
- `pkg/commitkit/commitkit.go:186-213` -- `SyncSubmoduleRef`; `git.StageFiles` at :192; whole-repo `git.HasStagedChanges` at :197; unscoped `git.Commit` at :208. Matches review exactly.
- `cmd/camp/refs/commands.go:57-62` -- staged-clean precheck (force guard); `executor.Stage(ctx, toSync)` at :94 (review cited :94-131; staging is :94, commit is :103, the loop detection block is :64-108).
- `internal/git/commit/commit.go:83-107` -- `stageAndCommit`, used as the engine; see task 01.

### Step 1: Fix syncParentRef (cmd/camp/project/commit.go:190-216)

The current code stages `relPath` then commits the whole repo without scoping:

```go
// Current (project/commit.go:196-212):
if err := parentExec.Stage(ctx, []string{relPath}); err != nil {
    return camperrors.Wrap(err, "staging submodule ref")
}
...
opts := &git.CommitOptions{Message: msg}
if err := parentExec.Commit(ctx, opts); err != nil { ... }
```

Replace with a scoped commit using the new engine. Since `syncParentRef` works at the campaign root (which is a plain git repo, not a submodule), `BuildTempIndexPath` resolves the gitdir correctly. Use the Mode A flow (populate temp index from HEAD, add the gitlink, commit):

```go
func syncParentRef(ctx context.Context, campRoot, relPath string, cfg *config.CampaignConfig) error {
    // Check whether the gitlink has actually changed before doing anything.
    hasRefChange, err := git.HasStagedPathChange(ctx, campRoot, relPath)
    if err != nil {
        return camperrors.Wrap(err, "check staged ref")
    }
    if !hasRefChange {
        // Stage the ref to see if there is a diff vs HEAD.
        if err := git.StageFiles(ctx, campRoot, relPath); err != nil {
            return camperrors.Wrap(err, "staging submodule ref")
        }
        hasRefChange, err = git.HasStagedPathChange(ctx, campRoot, relPath)
        if err != nil {
            return camperrors.Wrap(err, "check staged ref after add")
        }
        if !hasRefChange {
            return nil // no-op
        }
    }

    projName := filepath.Base(relPath)
    msg := fmt.Sprintf("update %s submodule ref", projName)
    if cfg != nil {
        msg = git.PrependCampaignTag(cfg.ID, msg)
    }

    // Use the scoped commit engine: populate temp index with only this gitlink.
    opts := commit.Options{
        CampaignRoot: campRoot,
        Files:        []string{relPath},
    }
    if cfg != nil {
        opts.CampaignID = cfg.ID
    }
    if err := commitScopedWithMessage(ctx, opts, msg); err != nil {
        if errors.Is(err, git.ErrNoChanges) {
            return nil
        }
        return camperrors.Wrap(err, "commit")
    }

    fmt.Println(ui.Success("✓ Campaign root synced (" + relPath + ")"))
    return nil
}
```

`HasStagedPathChange` is a new helper (add to `internal/git/commit.go`):

```go
// HasStagedPathChange reports whether path has staged changes relative to HEAD.
// Unlike HasStagedChanges (whole-repo), this is scoped to a single path.
func HasStagedPathChange(ctx context.Context, repoPath, path string) (bool, error) {
    cmd := exec.CommandContext(ctx, "git", "-C", repoPath, "diff", "--cached", "--quiet", "--", path)
    err := cmd.Run()
    if err != nil {
        if exitErr, ok := err.(*exec.ExitError); ok && exitErr.ExitCode() == 1 {
            return true, nil
        }
        return false, camperrors.NewGit("diff --cached path", "", "", "", err)
    }
    return false, nil
}
```

`commitScopedWithMessage` is a thin wrapper that calls `stageAndCommit` directly with a pre-built message string, bypassing the `doCommit` tag-building path (since `syncParentRef` builds its own message). Or, simply call `stageAndCommit` from the `commit` package with `opts.Files = []string{relPath}` and the pre-built `msg`. Keep the campaign tag building as-is in `syncParentRef`.

### Step 2: Fix SyncSubmoduleRef (pkg/commitkit/commitkit.go:186-213)

This is fest's public API (C-5). The signature stays identical. Only the internal implementation changes:

```go
// SyncSubmoduleRef stages the updated submodule pointer for projectRelPath
// inside the campaign root and commits it with a campaign-tagged message.
// projectRelPath is the path to the submodule relative to campaignRoot
// (e.g. "projects/fest").
//
// Semantic change (CH0001): the commit is now scoped to projectRelPath only.
// The no-op check is path-scoped (not whole-repo). Pre-staged changes at the
// campaign root that are unrelated to projectRelPath are preserved untouched.
//
// It is a no-op and returns nil when the submodule pointer has not changed.
//
// Uses automatic lock retry with stale lock cleanup.
func SyncSubmoduleRef(ctx context.Context, campaignRoot, projectRelPath, campaignID string) error {
    if ctx.Err() != nil {
        return ctx.Err()
    }

    // Stage only the submodule ref.
    if err := git.StageFiles(ctx, campaignRoot, projectRelPath); err != nil {
        return fmt.Errorf("commitkit: stage submodule %s: %w", projectRelPath, err)
    }

    // Path-scoped no-op check: only look at whether THIS ref has changed.
    hasChange, err := git.HasStagedPathChange(ctx, campaignRoot, projectRelPath)
    if err != nil {
        return fmt.Errorf("commitkit: check staged ref: %w", err)
    }
    if !hasChange {
        return nil // No-op: submodule pointer hasn't changed.
    }

    msg := git.PrependCampaignTag(campaignID,
        fmt.Sprintf("sync submodule ref: %s", projectRelPath))

    // Scoped commit: only projectRelPath is committed; unrelated staged content
    // at campaignRoot is left in the index.
    if err := git.CommitScoped(ctx, campaignRoot, []string{projectRelPath}, &git.CommitOptions{Message: msg}); err != nil {
        return fmt.Errorf("commitkit: commit submodule ref for %s: %w", projectRelPath, err)
    }

    return nil
}
```

`git.CommitScoped` is a new function in `internal/git/commit.go` that wraps the temp-index Mode A flow for a pre-staged path set:

```go
// CommitScoped creates a commit using a temp index populated with only the
// given paths. It is the safe alternative to Commit with Only: for any use
// case where the paths are file content (not just gitlinks). For submodule
// gitlinks, CommitScoped and git commit --only are equivalent because gitlink
// state IS the submodule HEAD.
//
// The lock-retry wrapper applies. The temp index is cleaned up on all paths.
func CommitScoped(ctx context.Context, repoPath string, paths []string, opts *CommitOptions) error {
    if opts == nil {
        return ErrCommitOptionsRequired
    }
    if err := opts.Validate(); err != nil {
        return err
    }
    if len(paths) == 0 {
        return ErrNoChanges
    }

    tmpPath, _, err := BuildTempIndexPath(repoPath)
    if err != nil {
        return err
    }
    defer RemoveTempIndex(tmpPath)

    if err := ReadTreeIntoTempIndex(ctx, repoPath, tmpPath); err != nil {
        return err
    }
    if err := AddPathsToTempIndex(ctx, repoPath, tmpPath, paths); err != nil {
        return err
    }

    opts.TempIndexPath = tmpPath
    return Commit(ctx, repoPath, opts)
}
```

### Step 3: Fix refs-sync (cmd/camp/refs/commands.go)

The current code stages all of `toSync` then commits without scoping (`:94-104`). Replace with `CommitScoped`. Also add printed output for skipped submodules.

The `detectRefChanges` loop (`:111-143`) silently `continue`s when `ls-tree` or `rev-parse` fail. Each continue represents a skipped submodule. Collect skip reasons:

```go
type refSkip struct {
    Path   string
    Reason string
}

func detectRefChanges(ctx context.Context, campRoot string, paths []string) ([]refChange, []refSkip, error) {
    var changes []refChange
    var skips []refSkip
    for _, p := range paths {
        fullPath := filepath.Join(campRoot, p)

        lsTreeOut, err := exec.CommandContext(ctx, "git", "-C", campRoot,
            "ls-tree", "HEAD", "--", p).Output()
        if err != nil {
            skips = append(skips, refSkip{Path: p, Reason: "ls-tree failed: " + err.Error()})
            continue
        }
        // ... parse fields ...
        headOut, err := exec.CommandContext(ctx, "git", "-C", fullPath,
            "rev-parse", "HEAD").Output()
        if err != nil {
            skips = append(skips, refSkip{Path: p, Reason: "rev-parse HEAD failed (submodule not checked out?): " + err.Error()})
            continue
        }
        // ... build refChange ...
    }
    return changes, skips, nil
}
```

In `runRefsSync`, after `detectRefChanges`, print skips:

```go
changes, skips, err := detectRefChanges(ctx, campRoot, paths)
if err != nil {
    return err
}

displayRefPlan(changes)

if len(skips) > 0 {
    fmt.Println(ui.Dim("Skipped submodules:"))
    for _, s := range skips {
        fmt.Printf("  %-35s %s\n", s.Path, ui.Dim(s.Reason))
    }
    fmt.Println()
}
```

Replace the staging+commit block:

```go
// Before (:94-104):
executor, err := git.NewExecutor(campRoot)
...
if err := executor.Stage(ctx, toSync); err != nil { ... }
...
if err := executor.Commit(ctx, &git.CommitOptions{Message: msg}); err != nil { ... }

// After:
if err := git.CommitScoped(ctx, campRoot, toSync, &git.CommitOptions{Message: msg}); err != nil {
    return camperrors.Wrap(err, "commit")
}
```

The staged-clean precheck at :57-62 (the `--force` guard) can remain as-is or be tightened to check only `toSync` paths. Tightening is a behavior improvement: the current check refuses if ANY staged content exists at the campaign root, which blocks refs-sync when the user has legit staged content. Consider replacing it with a per-path check: if any path in `toSync` is already staged (and the staged content differs from what refs-sync would write), warn but continue; if not `--force`, refuse. This is a UX improvement beyond the minimum fix. At minimum, keep the existing behavior and note the opportunity in a comment.

### Container/host test: pre-staged file survives sync

Add to `tests/integration/` (testcontainers, `//go:build integration`):

```
setup:
  campaign root with two submodules: A and B
  stage an unrelated file at the campaign root: echo "hello" > notes.txt; git add notes.txt

test camp p commit --sync (via submodule A):
  update submodule A to a new commit
  run camp project commit --sync from submodule A
  assert: notes.txt is still staged at campaign root (not swept into the sync commit)
  assert: git show HEAD -- projects/A shows the updated gitlink
  assert: git show HEAD -- notes.txt fails (notes.txt was not committed)

test SyncSubmoduleRef directly:
  call commitkit.SyncSubmoduleRef(ctx, campRoot, "projects/B", campaignID)
  assert: notes.txt is still staged at campaign root
  assert: git log --oneline -1 shows the submodule sync commit for projects/B only
```

## Done When

- [ ] All requirements met
- [ ] `syncParentRef` uses `HasStagedPathChange` for its no-op guard and `CommitScoped` (or equivalent) for the commit
- [ ] `SyncSubmoduleRef` uses `HasStagedPathChange` and `CommitScoped`; package-level doc comment updated
- [ ] `commitkit.SyncSubmoduleRef` signature is identical to before
- [ ] `refs-sync` commits scoped to `toSync` and prints skipped entries with reasons
- [ ] Container test passes: pre-staged unrelated file survives both `camp p commit --sync` and `SyncSubmoduleRef`
- [ ] Both-profile gate passes (`just build`, `BUILD_TAGS=dev just build`, `just test unit`, `go test -tags dev ./...`, `just lint`, `BUILD_TAGS=dev just lint`)
- [ ] `just test integration` passes with the new container test

## Out of Scope

- fest repo: no edits (C-5). The required mirror change to `fest/internal/commands/commit/commit.go:541-553` is documented as a workitem in task 05.
- `--force` guard semantic change in refs-sync: the minimum fix is scoped commit; the per-path guard improvement is optional and must not block this task.
