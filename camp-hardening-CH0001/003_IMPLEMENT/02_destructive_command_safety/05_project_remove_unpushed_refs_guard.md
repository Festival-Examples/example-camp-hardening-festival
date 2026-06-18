---
fest_type: task
fest_id: 05_project_remove_unpushed_refs_guard.md
fest_name: project_remove_unpushed_refs_guard
fest_parent: 02_destructive_command_safety
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.620285-06:00
fest_updated: 2026-06-13T12:33:10.695274-06:00
fest_tracking: true
---


# Task: Project Remove Unpushed Refs Guard

## Objective

Before deleting `.git/modules/projects/<name>`, detect refs that exist in the module gitdir but are not reachable from any remote; refuse the deletion without `--force`, naming the unreachable branches; and fix the help text that falsely claims project files remain in place after removal (N-3).

## Requirements

- [ ] `Remove` in `internal/project/remove.go` calls a new helper `detectUnpushedRefs(ctx, modulesPath)` before the `os.RemoveAll(modulesPath)` call at line 172; if unpushed refs are found and `opts.Force` is false, the function returns an error naming the branches.
- [ ] `detectUnpushedRefs` uses `git -C <modulesPath> for-each-ref --format=%(refname:short) refs/heads/` to enumerate local branches, then for each branch determines whether any commit exists that is not reachable from any remote tracking ref (using `git -C <modulesPath> log --oneline <branch> --not --remotes`); branches with at least one such commit are considered unpushed.
- [ ] When unpushed refs are found and `opts.Force` is false, the error message names each unpushed branch and explains that the module gitdir will be deleted; the user is told to push or pass `--force`.
- [ ] When `opts.Force` is true, the unpushed-refs check is skipped and the removal proceeds unchanged.
- [ ] The `Long` help text of `camp project remove` is corrected to remove the claim that project files remain in place (because `git rm -f` removes the project directory from the working tree; only `--delete` is needed to additionally remove the gitlink entry and worktrees).
- [ ] Container integration tests in `tests/integration/project_remove_unpushed_test.go` cover: remove refuses when unpushed branch exists; remove proceeds with `--force` despite unpushed branch; remove succeeds cleanly when all branches are pushed.

## Implementation

### Background: the bug mechanics

`internal/project/remove.go:167-175` (verified in worktree):

```go
// Always clean up .git/modules after a successful submodule removal,
// regardless of whether --delete was requested. Leaving stale module
// metadata causes re-add failures.
modulesPath := filepath.Join(campaignRoot, ".git", "modules", "projects", name)
if boundErr := pathutil.ValidateBoundary(campaignRoot, modulesPath); boundErr == nil {
    if removeErr := os.RemoveAll(modulesPath); removeErr == nil {
        addStep(result, fmt.Sprintf("removed .git/modules/projects/%s", name))
    }
}
```

The comment explains the motivation (re-add failures without cleanup), but the implementation deletes ALL local git history for the submodule. After `git submodule deinit -f` and `git rm -f`, the project directory is gone from the working tree and the gitlink entry is removed from the index. However, the `.git/modules/projects/<name>` directory is a full bare-style git repository that still holds all local branches, commits, and tags. If the user had a branch with committed-but-not-pushed work, all of that is permanently lost when this directory is removed.

Git intentionally preserves `.git/modules/<name>` on `submodule deinit` and `git rm` precisely so that local history survives project removal.

The `isDirtySubmodule` check at lines 147-158 runs `git status --porcelain` in the project directory and only catches UNCOMMITTED changes. A branch with committed work that is not pushed passes this check cleanly.

### Step 1: Write the `detectUnpushedRefs` helper

Add to `internal/project/remove.go`:

```go
// detectUnpushedRefs returns local branch names whose tips contain commits
// not reachable from any remote tracking ref. An empty list means all local
// branches are fully pushed (or there are no local branches).
//
// gitDir must be the path to the module's git directory
// (e.g., .git/modules/projects/<name>). The function runs git commands
// with --git-dir set to this path rather than relying on the working tree.
func detectUnpushedRefs(ctx context.Context, gitDir string) ([]string, error) {
    // List all local branches in the module gitdir.
    listCmd := exec.CommandContext(ctx, "git",
        "--git-dir", gitDir,
        "for-each-ref", "--format=%(refname:short)", "refs/heads/")
    listOutput, err := listCmd.Output()
    if err != nil {
        // If for-each-ref fails (e.g., not a git repo), treat as no branches.
        return nil, nil
    }

    var unpushed []string
    for _, branch := range strings.Split(strings.TrimSpace(string(listOutput)), "\n") {
        branch = strings.TrimSpace(branch)
        if branch == "" {
            continue
        }

        // Check whether this branch has any commit not reachable from remotes.
        // `git log --oneline <branch> --not --remotes` lists commits on branch
        // that do not exist on any remote tracking ref. If the output is
        // non-empty, the branch has unpushed commits.
        logCmd := exec.CommandContext(ctx, "git",
            "--git-dir", gitDir,
            "log", "--oneline", branch, "--not", "--remotes")
        logOutput, err := logCmd.Output()
        if err != nil {
            // A log failure (e.g., no remotes configured) means we cannot
            // confirm safety; treat as potentially unpushed to be conservative.
            unpushed = append(unpushed, branch)
            continue
        }

        if strings.TrimSpace(string(logOutput)) != "" {
            unpushed = append(unpushed, branch)
        }
    }
    return unpushed, nil
}
```

### Step 2: Insert the check before `os.RemoveAll` in `Remove`

Replace the existing module-removal block (lines 167-175):

```go
modulesPath := filepath.Join(campaignRoot, ".git", "modules", "projects", name)
if boundErr := pathutil.ValidateBoundary(campaignRoot, modulesPath); boundErr == nil {
    if removeErr := os.RemoveAll(modulesPath); removeErr == nil {
        addStep(result, fmt.Sprintf("removed .git/modules/projects/%s", name))
    }
}
```

With:

```go
modulesPath := filepath.Join(campaignRoot, ".git", "modules", "projects", name)
if boundErr := pathutil.ValidateBoundary(campaignRoot, modulesPath); boundErr == nil {
    // Guard: refuse to destroy local history when unpushed branches exist.
    if !opts.Force {
        unpushed, checkErr := detectUnpushedRefs(ctx, modulesPath)
        if checkErr == nil && len(unpushed) > 0 {
            return result, camperrors.Wrapf(
                camperrors.ErrInvalidInput,
                "project %q has %d unpushed branch(es) in .git/modules: %s\n"+
                    "push these branches or pass --force to discard local history",
                name, len(unpushed), strings.Join(unpushed, ", "),
            )
        }
    }
    if removeErr := os.RemoveAll(modulesPath); removeErr == nil {
        addStep(result, fmt.Sprintf("removed .git/modules/projects/%s", name))
    }
}
```

Import `"strings"` if not already in the import block.

### Step 3: Fix the help text

Locate the `camp project remove` command (the `Long` field of the cobra command). Find the sentence claiming files remain in place, for example:

> project files remain in place for you to handle manually

This is false: `git rm -f` removes the project directory from the working tree. The correct statement is:

> The project directory is removed from the working tree by git rm. Pass --delete to also remove any worktree directories managed by camp.

Update the `Long` or `Example` field accordingly. Find the exact command file by grepping:

```bash
grep -rn "remain in place\|files remain" cmd/camp/project/
```

### Edge cases

**Module gitdir does not exist:** `os.Stat(modulesPath)` is not called before the guard because the existing code only calls `os.RemoveAll` when `ValidateBoundary` passes; if the path doesn't exist, `for-each-ref` exits with an error and `detectUnpushedRefs` returns nil (treated as no branches, removal proceeds). This is safe.

**No remotes configured in the module gitdir:** `git log --not --remotes` with no remotes lists all commits as unpushed. This is the conservative-correct behavior: if there is no remote, every commit is genuinely unpushed. The error message will list the branches; the user can pass `--force` if they know they do not care.

**Module gitdir on a different filesystem (cross-device):** `exec.CommandContext` with `--git-dir` does not change directory and has no cross-device concern. The `--git-dir` flag tells git where the repository metadata lives; the commands run in the camp process's working directory.

**macOS `/private/var` symlinks:** `modulesPath` is constructed from `campaignRoot` and `filepath.Join`; both use the path as-is. The `--git-dir` flag accepts whatever path is given; git resolves symlinks internally. No special handling needed.

**`--dry-run`:** the dry-run branch at line 123 returns before reaching the submodule removal block, so `detectUnpushedRefs` is never called during dry runs. This is correct.

### Container integration tests

Create `tests/integration/project_remove_unpushed_test.go`:

```go
//go:build integration
// +build integration

package integration

import (
    "fmt"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func setupRemoveSubmoduleCampaign(t *testing.T, tc *TestContainer, name string) (campPath, projPath, barePath string) {
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
printf '# Proj\n' > %[2]s/README.md
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
GIT_ALLOW_PROTOCOL=file git submodule add %[2]s projects/subproj
git -C %[1]s commit -m 'add subproj'
`, campPath, barePath))

    projPath = campPath + "/projects/subproj"
    return campPath, projPath, barePath
}

func TestProjectRemove_UnpushedBranch_Refuses(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupRemoveSubmoduleCampaign(t, tc, "remove-unpushed")

    // Create a branch with a commit and do NOT push it.
    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b unpushed-branch
printf 'secret work\n' > %[1]s/secret.txt
git -C %[1]s add secret.txt && git -C %[1]s commit -m 'unpushed commit'
git -C %[1]s checkout main
`, projPath))

    // Remove without --force should refuse.
    _, err := tc.RunCampInDir(campPath, "project", "remove", "subproj")
    assert.Error(t, err, "should refuse when unpushed branch exists")
    assert.Contains(t, err.Error(), "unpushed", "error should mention unpushed branches")
    assert.Contains(t, err.Error(), "unpushed-branch", "error should name the branch")

    // The .git/modules entry must still exist.
    exists, err := tc.CheckDirExists(campPath + "/.git/modules/projects/subproj")
    require.NoError(t, err)
    assert.True(t, exists, ".git/modules entry must not be deleted on refusal")
}

func TestProjectRemove_UnpushedBranch_ForceSucceeds(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupRemoveSubmoduleCampaign(t, tc, "remove-force")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b force-branch
printf 'will lose\n' > %[1]s/loseme.txt
git -C %[1]s add loseme.txt && git -C %[1]s commit -m 'will be lost'
git -C %[1]s checkout main
`, projPath))

    _, err := tc.RunCampInDir(campPath, "project", "remove", "subproj", "--force")
    require.NoError(t, err, "remove --force should succeed even with unpushed branch")

    exists, err := tc.CheckDirExists(campPath + "/.git/modules/projects/subproj")
    require.NoError(t, err)
    assert.False(t, exists, ".git/modules entry should be removed with --force")
}

func TestProjectRemove_AllPushed_Succeeds(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath, _ := setupRemoveSubmoduleCampaign(t, tc, "remove-pushed")

    // Create a branch, push it, then remove the project.
    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s checkout -b pushed-branch
printf 'pushed\n' > %[1]s/pushed.txt
git -C %[1]s add pushed.txt && git -C %[1]s commit -m 'pushed commit'
GIT_ALLOW_PROTOCOL=file git -C %[1]s push origin pushed-branch
git -C %[1]s checkout main
`, projPath))

    _, err := tc.RunCampInDir(campPath, "project", "remove", "subproj")
    require.NoError(t, err, "remove should succeed when all branches are pushed")

    exists, err := tc.CheckDirExists(campPath + "/.git/modules/projects/subproj")
    require.NoError(t, err)
    assert.False(t, exists, ".git/modules entry should be cleaned up after successful remove")
}
```

Note: the `setupRemoveSubmoduleCampaign` helper name must not conflict with any existing helper in the integration suite. Check `helpers.go` and `project_remote_submodule_test.go` before adding. Rename if needed.

## Done When

- [ ] All requirements met
- [ ] `just build` and `BUILD_TAGS=dev just build` pass from the worktree root
- [ ] `just test unit` and `go test -tags dev ./...` pass
- [ ] `just lint` and `BUILD_TAGS=dev just lint` pass
- [ ] `just test integration` passes with the new `TestProjectRemove_*` tests
- [ ] `grep -n "remain in place\|files remain" cmd/camp/project/` returns no results from the remove command's help text
- [ ] Manual verification: `camp project remove` on a project with an unpushed branch prints the branch name and exits non-zero
- [ ] Manual verification: `camp project remove --force` on the same project succeeds

## Out of Scope

- `camp project remove --delete` (deletes project files) is a separate code path that runs after the submodule removal; the `.git/modules` guard applies to both paths because it precedes the `opts.Delete` block.
- The `isDirtySubmodule` check is not modified; it correctly catches uncommitted working-tree changes. This task adds the complementary check for committed-but-unpushed changes.
- `buildRecoveryInstructions` at line 251 already generates `rm -rf .git/modules/...` as a manual recovery step; it should remain unchanged since it is only shown when removal fails partway through.