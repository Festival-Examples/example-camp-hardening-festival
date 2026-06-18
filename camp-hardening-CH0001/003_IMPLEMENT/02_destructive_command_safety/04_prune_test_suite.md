---
fest_type: task
fest_id: 04_prune_test_suite.md
fest_name: prune_test_suite
fest_parent: 02_destructive_command_safety
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.620091-06:00
fest_updated: 2026-06-12T14:00:03.214214-06:00
fest_tracking: true
---


# Task: Prune Test Suite

## Objective

Add table-driven unit tests for `internal/prune` candidate/forced/merged branch classification using an injected fake executor, and a container integration test for `camp project prune` including `--remote-delete` against a bare origin fixture, giving coverage to a 618-line zero-test package that already deletes remote branches in production (R16/TEST-4).

## Requirements

- [ ] `internal/prune` has at least one new test file (`prune_test.go`) with table-driven unit tests covering: merged branch pruned, unmerged branch skipped, gone-upstream branch treated as forced-delete candidate, dirty worktree skipped (from task 03), and `--dry-run` produces `StatusWouldDelete` instead of `StatusDeleted`.
- [ ] The unit tests use a fake/stub executor that replaces `git.MergedBranchesFromRef`, `git.GoneBranches`, `git.HasChanges`, `git.DeleteBranch`, and `git.DeleteBranchForce` without running real git commands; these tests follow the host `t.TempDir()` convention.
- [ ] If `internal/prune/prune.go` has no existing seam for injection, this task introduces a minimal seam (a struct with function-typed fields, or a narrow interface) that replaces the direct `git.*` calls in the classification and deletion paths WITHOUT changing observable behavior; `just test unit` must still pass after the refactor.
- [ ] A container integration test in `tests/integration/prune_test.go` covers: `camp project prune` deletes a merged branch, skips an unmerged branch, skips a dirty worktree branch, and `--remote-delete` deletes the merged branch from a bare origin while leaving the unmerged remote branch.
- [ ] The container test uses a bare origin fixture (same pattern as `fresh_test.go` and `project_remote_submodule_test.go`).
- [ ] `deleteRemoteBranches` at `internal/prune/prune.go:541` (verified in worktree) is exercised by the `--remote-delete` container test.

## Implementation

### Background: the risk profile

`internal/prune` is 618 lines of logic that:
- Classifies local branches as merged, gone-upstream, or safe via `git.MergedBranchesFromRef` and `git.GoneBranches`
- Removes merged-branch worktrees via `wt.Remove` with `force=true` (fixed in task 03)
- Deletes local branches via `git.DeleteBranch` / `git.DeleteBranchForce`
- Pushes remote branch deletions via `git.DeleteRemoteBranch` (inside `deleteRemoteBranches` at line 541)

Zero tests exist anywhere in the package. The only adjacent coverage is `tests/integration/fresh_test.go` which exercises `camp fresh` end-to-end but does not isolate the prune classification logic.

### Step 1: Introduce the executor seam

Read through `internal/prune/prune.go` to identify all `git.*` function calls in the classification and deletion paths. In the verified worktree the key calls are:

- `git.DefaultBranch(ctx, path)` -- called in `Execute`
- `git.FetchRemotePrune(ctx, path, remote)` -- optional refresh
- `git.MergedBranchesFromRef(ctx, path, baseRef)` -- produces the `merged` list
- `git.GoneBranches(ctx, path)` -- produces the `gone` list
- `git.MergeBase(ctx, path, baseRef, b)` -- used in `partitionGoneByMergeEquivalence`
- `git.IsAncestor(ctx, path, ...)` -- used in `partitionGoneByMergeEquivalence`
- `git.BasePatchIDSet(ctx, path, ...)` -- used in `partitionGoneByMergeEquivalence`
- `git.CumulativePatchID(ctx, path, ...)` -- used in `partitionGoneByMergeEquivalence`
- `git.HasChanges(ctx, path)` -- used in `detachedWorktreeClean` and (after task 03) in the branch worktree dirty check
- `git.DeleteBranch(ctx, path, branch)` -- in `deleteLocalBranches`
- `git.DeleteBranchForce(ctx, path, branch)` -- in `deleteLocalBranches`
- `git.DeleteRemoteBranch(ctx, path, branch)` -- in `deleteRemoteBranches`
- `git.PruneRemote(ctx, path)` -- in `pruneTrackingRefs`

The minimal seam needed for classification unit tests:

```go
// gitOps abstracts the git operations needed by Execute for testing.
// The production implementation calls the git package functions directly.
type gitOps interface {
    MergedBranches(ctx context.Context, path, baseRef string) ([]string, error)
    GoneBranches(ctx context.Context, path string) ([]string, error)
    DefaultBranch(ctx context.Context, path string) string
    MergeBase(ctx context.Context, path, ref1, ref2 string) (string, error)
    IsAncestor(ctx context.Context, path, a, b string) (bool, error)
    BasePatchIDSet(ctx context.Context, path, from, to string) (map[string]struct{}, error)
    CumulativePatchID(ctx context.Context, path, base, branch string) (string, error)
    HasChanges(ctx context.Context, path string) (bool, error)
    DeleteBranch(ctx context.Context, path, branch string) error
    DeleteBranchForce(ctx context.Context, path, branch string) error
    DeleteRemoteBranch(ctx context.Context, path, branch string) error
    FetchRemotePrune(ctx context.Context, path, remote string) error
    PruneRemote(ctx context.Context, path string) (int, error)
}
```

The production `realGitOps` struct implements this by delegating to the `git` package. Add a package-level variable for the production ops and an internal `executeWithOps` function:

```go
var productionOps gitOps = &realGitOps{}

// Execute runs the prune logic for a single project using the production
// git operations. Test code calls executeWithOps directly.
func Execute(ctx context.Context, name, path string, opts Options) ProjectResult {
    return executeWithOps(ctx, name, path, opts, productionOps)
}

func executeWithOps(ctx context.Context, name, path string, opts Options, ops gitOps) ProjectResult {
    // ... existing Execute body with git.X calls replaced by ops.X calls ...
}
```

Verify that `Execute`'s signature and behavior are unchanged by running `just test unit` (empty suite passes trivially but build verification is real). The refactor is strictly mechanical: same logic, same flow, `git.X` replaced by `ops.X`.

Note: `wt.List` and `wt.Remove` on the `worktree.GitWorktree` struct are separate from the `gitOps` interface. For unit tests of classification, the worktree listing can return nil (simulating no active worktrees) by using the fact that `detectWorktreesForBranches` calls `wt.List` on the actual path. Stub this by creating a temporary directory that contains no git metadata; `wt.List` will fail gracefully and return nil, which means `wtMap` is empty and no worktrees are involved in the test. This keeps the test footprint small.

### Step 2: Write the unit tests

Create `internal/prune/prune_test.go` using `t.TempDir()` convention (no git operations; fake ops return controlled lists):

```go
package prune

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

type fakeGitOps struct {
    merged     []string
    gone       []string
    hasChanges map[string]bool
    deleted    []string
    deleteErr  error
}

func (f *fakeGitOps) DefaultBranch(_ context.Context, _ string) string { return "main" }
func (f *fakeGitOps) MergedBranches(_ context.Context, _, _ string) ([]string, error) {
    return f.merged, nil
}
func (f *fakeGitOps) GoneBranches(_ context.Context, _ string) ([]string, error) {
    return f.gone, nil
}
func (f *fakeGitOps) MergeBase(_ context.Context, _, _, _ string) (string, error) {
    return "abc1234", nil
}
func (f *fakeGitOps) IsAncestor(_ context.Context, _, _, _ string) (bool, error) {
    return true, nil
}
func (f *fakeGitOps) BasePatchIDSet(_ context.Context, _, _, _ string) (map[string]struct{}, error) {
    return map[string]struct{}{}, nil
}
func (f *fakeGitOps) CumulativePatchID(_ context.Context, _, _, _ string) (string, error) {
    return "", nil
}
func (f *fakeGitOps) HasChanges(_ context.Context, path string) (bool, error) {
    return f.hasChanges[path], nil
}
func (f *fakeGitOps) DeleteBranch(_ context.Context, _, branch string) error {
    if f.deleteErr != nil {
        return f.deleteErr
    }
    f.deleted = append(f.deleted, branch)
    return nil
}
func (f *fakeGitOps) DeleteBranchForce(_ context.Context, _, branch string) error {
    if f.deleteErr != nil {
        return f.deleteErr
    }
    f.deleted = append(f.deleted, branch)
    return nil
}
func (f *fakeGitOps) DeleteRemoteBranch(_ context.Context, _, branch string) error { return nil }
func (f *fakeGitOps) FetchRemotePrune(_ context.Context, _, _ string) error        { return nil }
func (f *fakeGitOps) PruneRemote(_ context.Context, _ string) (int, error)         { return 0, nil }

func TestExecuteWithOps_MergedBranchDeleted(t *testing.T) {
    dir := t.TempDir()
    ops := &fakeGitOps{merged: []string{"feature/done"}}
    pr := executeWithOps(context.Background(), "proj", dir, Options{
        Force:   true,
        BaseRef: "main",
    }, ops)

    require.Empty(t, pr.Error)
    assert.Contains(t, ops.deleted, "feature/done")
    var found bool
    for _, r := range pr.Results {
        if r.Branch == "feature/done" && r.Status == StatusDeleted {
            found = true
        }
    }
    assert.True(t, found)
}

func TestExecuteWithOps_NoBranches_NoDeletions(t *testing.T) {
    dir := t.TempDir()
    ops := &fakeGitOps{}
    pr := executeWithOps(context.Background(), "proj", dir, Options{
        Force:   true,
        BaseRef: "main",
    }, ops)

    require.Empty(t, pr.Error)
    assert.Empty(t, ops.deleted)
}

func TestExecuteWithOps_DryRun_WouldDelete(t *testing.T) {
    dir := t.TempDir()
    ops := &fakeGitOps{merged: []string{"feature/dry"}}
    pr := executeWithOps(context.Background(), "proj", dir, Options{
        Force:   true,
        DryRun:  true,
        BaseRef: "main",
    }, ops)

    require.Empty(t, pr.Error)
    assert.Empty(t, ops.deleted, "dry-run must not call DeleteBranch")
    var found bool
    for _, r := range pr.Results {
        if r.Branch == "feature/dry" && r.Status == StatusWouldDelete {
            found = true
        }
    }
    assert.True(t, found, "dry-run should produce StatusWouldDelete")
}

func TestExecuteWithOps_GoneUpstreamUnsafe_Skipped(t *testing.T) {
    dir := t.TempDir()
    // fakeGitOps.CumulativePatchID returns "" and BasePatchIDSet returns empty map,
    // so the gone branch falls into unsafeGone and is skipped.
    ops := &fakeGitOps{gone: []string{"feature/gone"}}
    pr := executeWithOps(context.Background(), "proj", dir, Options{
        Force:   true,
        BaseRef: "main",
    }, ops)

    require.Empty(t, pr.Error)
    assert.Empty(t, ops.deleted)
    var found bool
    for _, r := range pr.Results {
        if r.Branch == "feature/gone" && r.Status == StatusSkipped {
            found = true
        }
    }
    assert.True(t, found, "unsafe gone branch must be skipped")
}
```

Note: `TestExecuteWithOps_DirtyWorktreeBranchSkipped` requires task 03's `DiscardDirty` field and the dirty-check integration into `deleteLocalBranches`. Once task 03 is complete, add:

```go
func TestExecuteWithOps_DirtyWorktree_Skipped(t *testing.T) {
    dir := t.TempDir()
    // This test requires a worktree seam or a real worktree entry in the dir.
    // Without a worktree seam, the worktree list returns nil (no worktrees),
    // so this specific path is tested in the container integration test instead.
    t.Skip("dirty-worktree unit coverage deferred to container test; see TestPrune_DirtyWorktreeBranchSkipped")
}
```

The container test below provides the real coverage.

### Step 3: Container integration test

Create `tests/integration/prune_test.go`:

```go
//go:build integration
// +build integration

package integration

import (
    "fmt"
    "strings"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func setupPruneCampaign(t *testing.T, tc *TestContainer, name string) (campPath, projPath, barePath string) {
    t.Helper()
    campPath = "/campaigns/" + name
    barePath = "/test/" + name + "-origin.git"
    seedDir := "/test/" + name + "-seed"

    tc.Shell(t, fmt.Sprintf(`
set -e
git init --bare %[1]s
git clone %[1]s %[2]s
git -C %[2]s config user.email test@test.com
git -C %[2]s config user.name Test
printf '# Project\n' > %[2]s/README.md
git -C %[2]s add . && git -C %[2]s commit -m 'init'
git -C %[2]s branch -M main
git -C %[2]s push origin main
git --git-dir %[1]s symbolic-ref HEAD refs/heads/main
`, barePath, seedDir))

    _, err := tc.InitCampaign(campPath, name, "product")
    require.NoError(t, err)

    tc.Shell(t, fmt.Sprintf(`
set -e
cd %[1]s
GIT_ALLOW_PROTOCOL=file git submodule add %[2]s projects/proj
git -C %[1]s commit -m 'add proj'
`, campPath, barePath))

    projPath = campPath + "/projects/proj"
    return campPath, projPath, barePath
}

func TestPrune_MergedBranchPruned(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupPruneCampaign(t, tc, "prune-merged")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b merged-feature
printf 'new\n' > %[1]s/feature.txt
git -C %[1]s add feature.txt && git -C %[1]s commit -m 'add feature'
git -C %[1]s checkout main
git -C %[1]s merge --no-ff merged-feature -m 'Merge merged-feature'
`, projPath))

    out, err := tc.RunCampInDir(campPath, "project", "prune", "--project", "proj", "--force")
    require.NoError(t, err)
    assert.Contains(t, strings.ToLower(out), "merged-feature")

    branchOut := tc.Shell(t, fmt.Sprintf(`git -C %s branch`, projPath))
    assert.NotContains(t, branchOut, "merged-feature")
}

func TestPrune_UnmergedBranchSkipped(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupPruneCampaign(t, tc, "prune-unmerged")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b unmerged-feature
printf 'wip\n' > %[1]s/wip.txt
git -C %[1]s add wip.txt && git -C %[1]s commit -m 'wip'
git -C %[1]s checkout main
`, projPath))

    _, err := tc.RunCampInDir(campPath, "project", "prune", "--project", "proj", "--force")
    require.NoError(t, err)

    branchOut := tc.Shell(t, fmt.Sprintf(`git -C %s branch`, projPath))
    assert.Contains(t, branchOut, "unmerged-feature", "unmerged branch must still exist")
}

func TestPrune_DirtyWorktreeBranchSkipped(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupPruneCampaign(t, tc, "prune-dirty-wt")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b dirty-wt-branch
printf 'content\n' > %[1]s/content.txt
git -C %[1]s add content.txt && git -C %[1]s commit -m 'content'
git -C %[1]s checkout main
git -C %[1]s merge --no-ff dirty-wt-branch -m 'Merge dirty-wt-branch'
git -C %[1]s worktree add %[2]s/projects/worktrees/proj/dirty-wt-branch dirty-wt-branch
printf 'uncommitted\n' > %[2]s/projects/worktrees/proj/dirty-wt-branch/uncommitted.txt
`, projPath, campPath))

    out, err := tc.RunCampInDir(campPath, "project", "prune", "--project", "proj", "--force")
    require.NoError(t, err)
    assert.Contains(t, strings.ToLower(out), "dirty", "output should mention dirty worktree")

    exists, err := tc.CheckDirExists(campPath + "/projects/worktrees/proj/dirty-wt-branch")
    require.NoError(t, err)
    assert.True(t, exists, "dirty worktree must not be removed")
}

func TestPrune_RemoteDelete_MergedBranchDeletedOnOrigin(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, barePath := setupPruneCampaign(t, tc, "prune-remote-del")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b remote-feature
printf 'remote\n' > %[1]s/remote.txt
git -C %[1]s add remote.txt && git -C %[1]s commit -m 'remote feature'
GIT_ALLOW_PROTOCOL=file git -C %[1]s push origin remote-feature
git -C %[1]s checkout main
git -C %[1]s merge --no-ff remote-feature -m 'Merge remote-feature'
GIT_ALLOW_PROTOCOL=file git -C %[1]s fetch --prune origin
`, projPath))

    // deleteRemoteBranches prompts for [y/N]; pipe "y" via shell to simulate confirmation.
    tc.Shell(t, fmt.Sprintf(
        `cd %s && printf 'y\n' | /camp project prune --project proj --force --remote-delete`,
        campPath))

    remoteOut := tc.Shell(t, fmt.Sprintf(`git --git-dir %s branch`, barePath))
    assert.NotContains(t, remoteOut, "remote-feature", "merged branch must be deleted from origin")
}

func TestPrune_DryRun_NoDeletions(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupPruneCampaign(t, tc, "prune-dryrun")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b dry-branch
printf 'dry\n' > %[1]s/dry.txt
git -C %[1]s add dry.txt && git -C %[1]s commit -m 'dry'
git -C %[1]s checkout main
git -C %[1]s merge --no-ff dry-branch -m 'Merge dry-branch'
`, projPath))

    out, err := tc.RunCampInDir(campPath, "project", "prune", "--project", "proj",
        "--dry-run", "--force")
    require.NoError(t, err)
    _ = out

    branchOut := tc.Shell(t, fmt.Sprintf(`git -C %s branch`, projPath))
    assert.Contains(t, branchOut, "dry-branch", "dry-run must not delete branches")
}
```

### Edge cases

**Interactive prompt in container tests:** container exec is non-interactive (not a TTY). The prune command currently prompts for `[y/N]` when `!opts.Force`. All container test calls pass `--force` to bypass the local-branch-deletion prompt. The `--remote-delete` test pipes `"y\n"` via shell because the remote-deletion prompt runs independently of `--force` (this is a documented gap; deferred to sequence 10).

**`setupPruneCampaign` helper name:** if `fresh_test.go` already defines a helper with this name, rename it to `setupPruneCampaign` locally or factor the common bare-origin setup into a shared helper in `helpers.go`.

## Done When

- [ ] All requirements met
- [ ] `just build` and `BUILD_TAGS=dev just build` pass from the worktree root
- [ ] `just test unit` and `go test -tags dev ./...` pass, including the new `internal/prune/prune_test.go`
- [ ] `just test integration` passes with the new `TestPrune_*` tests in `tests/integration/prune_test.go`
- [ ] `just lint` and `BUILD_TAGS=dev just lint` pass
- [ ] `go test -count=1 ./internal/prune/...` runs at least 4 test functions

## Out of Scope

- `deleteRemoteBranches` non-TTY prompt bypass (documented gap, deferred to sequence 10 CLI consistency work).
- `internal/commands/flow` tests are TEST-4's third ranked item and belong in sequence 04_gate_enforcement.
- `internal/workitem/selector` tests are TEST-4's fourth ranked item and belong in sequence 04 or 11.