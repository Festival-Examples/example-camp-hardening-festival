---
fest_type: task
fest_id: 01_worktrees_clean_guards.md
fest_name: worktrees_clean_guards
fest_parent: 02_destructive_command_safety
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.61895-06:00
fest_updated: 2026-06-12T13:45:47.435305-06:00
fest_tracking: true
---


# Task: Worktrees Clean Guards

## Objective

Add a confirmation prompt, restrict `os.RemoveAll` to the "gitdir target does not exist" staleness reason only, refuse to auto-remove `.git`-directory entries, close the empty-projectPath bypass, and require a dirty-tree check when `--force` is used -- eliminating the `camp worktrees clean` data-loss path (R2/GIT-1 plus second-pass correction 1).

## Requirements

- [ ] `os.RemoveAll` is called only when `checkWorktreeStale` returned `"gitdir target does not exist"`; all other staleness reasons skip filesystem removal.
- [ ] Entries where `.git` is a directory (staleness reason `".git is a directory (not a worktree)"`) are listed as skipped with a human-readable guidance message; they are never auto-removed regardless of `--force`.
- [ ] An interactive confirmation prompt appears before any deletion when stdin is a terminal and neither `--yes` nor `--force` is present; non-TTY invocations without `--yes`/`--force` refuse with a non-zero exit.
- [ ] When `projectPath` is empty (project not found in campaign config), the function does NOT fall through to `os.RemoveAll`; it returns an error naming the missing project.
- [ ] When `--force` is passed and the target is a real worktree (not the gitdir-missing case), `git status` runs in the worktree directory; a dirty tree causes the entry to be skipped with a clear message unless `--discard-dirty` is also passed.
- [ ] `--dry-run` continues to preview without making changes, and the preview output clearly distinguishes which entries would be removed versus skipped.
- [ ] Container integration tests in `tests/integration/worktrees_clean_test.go` cover: dirty worktree skipped, `.git`-directory entry never removed, truly stale entry removed, empty-projectPath case, and `--dry-run`.

## Implementation

### Background: the bug mechanics

The current `cleanWorktree` function in `cmd/camp/worktrees/clean.go` has two independent data-loss paths.

**Path A (line 238-243): git.Remove error swallow.** When `projectPath != ""`, the code calls `git.Remove` and silently ignores any error with `// Fall through to filesystem removal`. The error is not returned, not logged, and not inspected. `os.RemoveAll` at line 247 then always runs. This means `git worktree remove` refusing because the tree is dirty is completely ignored and the directory is deleted anyway.

**Path B (lines 228-237): empty-projectPath bypasses git entirely.** When the project is not found in campaign config, `projectPath` stays `""`. The `if projectPath != ""` guard is false, so `git.Remove` is never called, and execution falls straight to `os.RemoveAll` at line 247. A worktree whose project was renamed or removed from config silently gets its directory deleted with no git involvement whatsoever.

The `checkWorktreeStale` function at lines 189-226 returns one of these reasons for a non-empty string:
- `"missing .git file"` - directory exists but no `.git` at all
- `"cannot access .git: <err>"` - permissions/I/O issue
- `".git is a directory (not a worktree)"` - this is a full repo clone, not a worktree
- `"cannot read .git file: <err>"`
- `"invalid .git file format"`
- `"gitdir target does not exist"` - the canonical "truly stale" case: `.git` file exists and points at a gitdir that is gone

Additionally, `runWorktreesClean` at lines 106-138 prints the stale list and immediately calls `cleanWorktree` in a loop with no confirmation prompt. Only `--dry-run` saves you from immediate deletion.

### Step 1: Add a `--yes` flag

In the `init()` block of `cmd/camp/worktrees/clean.go` (after line 59), add:

```go
worktreesCleanCmd.Flags().BoolVarP(&cleanYes, "yes", "y", false,
    "Skip confirmation prompt")
```

Add `cleanYes bool` to the `var` block at the top of the file (alongside `cleanForce`, `cleanDryRun`, etc.).

### Step 2: Classify staleness reasons into removal tiers

Add a helper that maps the staleness reason string to a removal decision:

```go
// stalenessAllowsRemoval returns true only for the case where the gitdir
// target is definitively gone -- the worktree is unrecoverable by git and
// safe to remove from the filesystem without a git operation.
// Every other staleness reason requires a git operation (or is non-removable).
func stalenessAllowsRemoval(reason string) bool {
    return reason == "gitdir target does not exist"
}

// stalenessIsGitDir returns true when the entry appears to be a full clone
// (has a .git directory) rather than a worktree. These must never be auto-removed.
func stalenessIsGitDir(reason string) bool {
    return reason == ".git is a directory (not a worktree)"
}
```

### Step 3: Rewrite `findStaleWorktrees` to carry the tier

Update `cleanResult` to include a `gitDirEntry bool` field:

```go
type cleanResult struct {
    project    string
    worktree   string
    path       string
    reason     string
    gitDirEntry bool   // true: .git is a directory; never auto-remove
    removed    bool
    removeErr  error
}
```

In `findStaleWorktrees`, populate the new field:

```go
stale = append(stale, cleanResult{
    project:     projectName,
    worktree:    name,
    path:        wtPath,
    reason:      reason,
    gitDirEntry: stalenessIsGitDir(reason),
})
```

### Step 4: Add confirmation prompt in `runWorktreesClean`

Replace the current "Cleaning up..." block (lines 119-138) with logic that first prints the categorized list, skips non-removable entries, then prompts before deleting:

```go
// Separate entries into removable and non-removable categories.
var toRemove, toSkip []cleanResult
for _, s := range stale {
    if s.gitDirEntry {
        toSkip = append(toSkip, s)
    } else {
        toRemove = append(toRemove, s)
    }
}

// Always print skipped entries with guidance.
if len(toSkip) > 0 {
    fmt.Printf("\nSkipping %d entries whose .git is a directory (not a worktree):\n", len(toSkip))
    for _, s := range toSkip {
        fmt.Printf("  %s/%s\n", s.project, s.worktree)
        fmt.Printf("    Path:   %s\n", ui.Dim(s.path))
        fmt.Printf("    Reason: %s. This looks like a full clone;\n", ui.Dim(s.reason))
        fmt.Printf("            remove it manually after verifying no uncommitted work.\n\n")
    }
}

if len(toRemove) == 0 {
    fmt.Println(ui.Info("No removable stale worktrees found"))
    return nil
}

if cleanDryRun {
    fmt.Println(ui.Info("Dry run - no changes made"))
    return nil
}

// Require confirmation when stdin is a terminal and no --yes/--force.
if !cleanYes && !cleanForce {
    if navtui.IsTerminal() {
        fmt.Printf("\nAbout to remove %d worktree director(ies). Proceed? [y/N] ", len(toRemove))
        var answer string
        fmt.Scanln(&answer)
        if !strings.HasPrefix(strings.ToLower(answer), "y") {
            fmt.Println(ui.Info("Aborted"))
            return nil
        }
    } else {
        return fmt.Errorf("refusing to delete worktrees without confirmation in non-interactive mode; pass --yes or --force")
    }
}
```

Import `navtui "github.com/Obedience-Corp/camp/internal/nav/tui"` and `"strings"` in the import block.

### Step 5: Rewrite `cleanWorktree` with correct semantics

Replace the entire `cleanWorktree` function body (lines 228-252) with:

```go
func cleanWorktree(ctx context.Context, cfg *config.CampaignConfig, resolver *paths.Resolver, result *cleanResult, force, discardDirty bool) error {
    // Entries with a gitdir pointing nowhere are the only ones we remove
    // without a git operation. The git object store is already gone so there
    // is nothing for git to track.
    if stalenessAllowsRemoval(result.reason) {
        if err := os.RemoveAll(result.path); err != nil {
            return camperrors.Wrap(err, "failed to remove directory")
        }
        return nil
    }

    // For all other staleness reasons we need to route through git so that
    // git's own safety checks (dirty tree, active HEAD) are respected.
    var projectPath string
    for _, proj := range cfg.Projects {
        if proj.Name == result.project {
            projectPath = resolver.Project(result.project)
            break
        }
    }

    if projectPath == "" {
        return camperrors.Wrapf(camperrors.ErrNotFound,
            "project %q not found in campaign config; cannot remove worktree safely without a project path",
            result.project)
    }

    // When force is requested on a real worktree, run a dirty check before
    // passing --force to git. git worktree remove --force bypasses git's own
    // safety checks, so we apply the check ourselves.
    if force {
        hasChanges, err := git.HasChanges(ctx, result.path)
        if err == nil && hasChanges && !discardDirty {
            return fmt.Errorf("worktree %s/%s has uncommitted changes; commit or stash first, or pass --discard-dirty to override",
                result.project, result.worktree)
        }
    }

    gw := worktree.NewGitWorktree(projectPath)
    if err := gw.Remove(ctx, result.path, force); err != nil {
        return camperrors.Wrap(err, "git worktree remove failed")
    }
    return nil
}
```

Update the call site in the loop to pass the new signature:

```go
err := cleanWorktree(ctx, cfg, resolver, &stale[i], cleanForce, cleanDiscardDirty)
```

Add `cleanDiscardDirty bool` to the `var` block and a corresponding flag:

```go
worktreesCleanCmd.Flags().BoolVar(&cleanDiscardDirty, "discard-dirty", false,
    "Allow removal of worktrees with uncommitted changes (requires --force)")
```

### Step 6: Update the loop to use only `toRemove`

Replace the `for i := range stale` loop body with a loop over `toRemove` only. The `toSkip` entries have already been printed in step 4.

### Edge cases

**macOS `/private/var` symlinks:** `result.path` is constructed by `pm.WorktreePath` which joins `resolver.Worktrees()` with the project and name. No EvalSymlinks is performed at this level. `git.HasChanges` runs `git status` inside the worktree directory; git itself resolves symlinks internally so this is safe. If you add any string-based prefix comparison here, use `filepath.EvalSymlinks` first.

**Non-TTY behavior:** the confirmation gate must check `navtui.IsTerminal()` before calling `fmt.Scanln`; in a container, piped CI, or agent invocation stdin is not a terminal and `fmt.Scanln` reads EOF silently, which would appear as "N". The explicit check + error message is clearer and matches the intent-add pattern in `cmd/camp/intent/add.go:159`.

**`--dry-run` with `--force`:** dry-run takes priority. No deletions occur; the dry-run output should indicate which entries would be skipped (`.git`-directory) versus removed.

### Step 7: Container integration tests

Create `tests/integration/worktrees_clean_test.go`:

```go
//go:build integration
// +build integration

package integration

import (
    "strings"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// setupWorktreeCleanCampaign creates a campaign with one submodule project
// and one registered worktree at the given path inside the container.
func setupWorktreeCleanCampaign(t *testing.T, tc *TestContainer, name string) (campPath, projPath string) {
    t.Helper()
    campPath = "/campaigns/" + name
    bareDir := "/test/" + name + "-origin.git"
    seedDir := "/test/" + name + "-seed"

    tc.Shell(t, fmt.Sprintf(`
set -e
git init --bare %[1]s
git clone %[1]s %[2]s
git -C %[2]s config user.email test@test.com
git -C %[2]s config user.name Test
printf '# Test\n' > %[2]s/README.md
git -C %[2]s add . && git -C %[2]s commit -m 'init'
git -C %[2]s branch -M main
git -C %[2]s push origin main
git --git-dir %[1]s symbolic-ref HEAD refs/heads/main
`, bareDir, seedDir))

    _, err := tc.InitCampaign(campPath, name, "product")
    require.NoError(t, err)

    tc.Shell(t, fmt.Sprintf(`
set -e
cd %[1]s
GIT_ALLOW_PROTOCOL=file git submodule add %[2]s projects/proj
git commit -m 'add proj'
`, campPath, bareDir))

    projPath = campPath + "/projects/proj"
    return campPath, projPath
}

func TestWorktreesClean_TrulyStalEntryRemoved(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath := setupWorktreeCleanCampaign(t, tc, "wt-clean-stale")

    // Create a worktree and then delete its gitdir manually to simulate stale state.
    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s worktree add %[2]s/projects/worktrees/proj/stale-branch -b stale-branch
GITDIR=$(cat %[2]s/projects/worktrees/proj/stale-branch/.git | sed 's/gitdir: //')
rm -rf $GITDIR
`, projPath, campPath))

    out, err := tc.RunCampInDir(campPath, "worktrees", "clean", "--all", "--yes")
    require.NoError(t, err)
    assert.Contains(t, out, "removed")

    exists, err := tc.CheckDirExists(campPath + "/projects/worktrees/proj/stale-branch")
    require.NoError(t, err)
    assert.False(t, exists, "stale worktree directory should be removed")
}

func TestWorktreesClean_DirtyWorktreeSkipped(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath := setupWorktreeCleanCampaign(t, tc, "wt-clean-dirty")

    // Create a worktree, then make the gitdir appear missing to trigger stale
    // detection but leave uncommitted files in the worktree directory.
    // Actually: create a valid worktree but corrupt the gitdir reference so
    // checkWorktreeStale returns "gitdir target does not exist" -- but the
    // worktree still has uncommitted changes.
    //
    // The safer test: create a worktree with a reason OTHER than
    // "gitdir target does not exist" (e.g. invalid .git format) and verify
    // it is not removed without --force.
    tc.Shell(t, fmt.Sprintf(`
set -e
mkdir -p %[1]s/projects/worktrees/proj/dirty-wt
printf 'gitdir: /nonexistent/path\n' > %[1]s/projects/worktrees/proj/dirty-wt/.git
printf 'uncommitted\n' > %[1]s/projects/worktrees/proj/dirty-wt/changes.txt
`, campPath))

    // Without --force, only gitdir-target-missing entries are auto-removed.
    // This entry has an invalid gitdir, not a missing one, so it stays.
    out, err := tc.RunCampInDir(campPath, "worktrees", "clean", "--all", "--yes")
    // Should not error but also should not remove since gitdir just doesn't exist now
    // The path created above has a gitdir pointing to /nonexistent -- that IS "gitdir target does not exist"
    // so it WILL be removed. Adjust: use "invalid .git file format" instead.
    _ = out
    _ = err

    // Correct setup: make the .git file have invalid format
    tc.Shell(t, fmt.Sprintf(`
set -e
mkdir -p %[1]s/projects/worktrees/proj/dirty-wt2
printf 'not-a-gitdir-file\n' > %[1]s/projects/worktrees/proj/dirty-wt2/.git
printf 'uncommitted\n' > %[1]s/projects/worktrees/proj/dirty-wt2/changes.txt
`, campPath))

    out2, err2 := tc.RunCampInDir(campPath, "worktrees", "clean", "--all", "--yes")
    require.NoError(t, err2)
    assert.NotContains(t, out2, "dirty-wt2", "invalid-format entry should not be auto-removed")

    exists, err := tc.CheckDirExists(campPath + "/projects/worktrees/proj/dirty-wt2")
    require.NoError(t, err)
    assert.True(t, exists, "non-gitdir-missing worktree should be preserved")
}

func TestWorktreesClean_GitDirEntryNeverRemoved(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, _ := setupWorktreeCleanCampaign(t, tc, "wt-clean-gitdir")

    // Create a directory with a .git directory (full clone, not a worktree).
    tc.Shell(t, fmt.Sprintf(`
set -e
git init %[1]s/projects/worktrees/proj/full-clone
git -C %[1]s/projects/worktrees/proj/full-clone config user.email t@t.com
git -C %[1]s/projects/worktrees/proj/full-clone config user.name T
printf '# clone\n' > %[1]s/projects/worktrees/proj/full-clone/README.md
git -C %[1]s/projects/worktrees/proj/full-clone add .
git -C %[1]s/projects/worktrees/proj/full-clone commit -m 'init clone'
`, campPath))

    out, err := tc.RunCampInDir(campPath, "worktrees", "clean", "--all", "--yes")
    require.NoError(t, err)
    assert.Contains(t, out, "Skipping", "output should mention the skipped git-dir entry")

    exists, err := tc.CheckDirExists(campPath + "/projects/worktrees/proj/full-clone")
    require.NoError(t, err)
    assert.True(t, exists, ".git-directory entry must not be removed")
}

func TestWorktreesClean_DryRunNoChanges(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath := setupWorktreeCleanCampaign(t, tc, "wt-clean-dryrun")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s worktree add %[2]s/projects/worktrees/proj/dry-branch -b dry-branch
GITDIR=$(cat %[2]s/projects/worktrees/proj/dry-branch/.git | sed 's/gitdir: //')
rm -rf $GITDIR
`, projPath, campPath))

    out, err := tc.RunCampInDir(campPath, "worktrees", "clean", "--all", "--dry-run")
    require.NoError(t, err)
    assert.Contains(t, out, "Dry run")

    exists, err := tc.CheckDirExists(campPath + "/projects/worktrees/proj/dry-branch")
    require.NoError(t, err)
    assert.True(t, exists, "--dry-run must not remove anything")
}

func TestWorktreesClean_NonTTY_RefusesWithoutFlag(t *testing.T) {
    tc := GetSharedContainer(t)
    campPath, projPath := setupWorktreeCleanCampaign(t, tc, "wt-clean-notty")

    tc.Shell(t, fmt.Sprintf(`
set -e
git -C %[1]s worktree add %[2]s/projects/worktrees/proj/notty-branch -b notty-branch
GITDIR=$(cat %[2]s/projects/worktrees/proj/notty-branch/.git | sed 's/gitdir: //')
rm -rf $GITDIR
`, projPath, campPath))

    // Container exec is not a TTY; without --yes this should refuse.
    _, err := tc.RunCampInDir(campPath, "worktrees", "clean", "--all")
    assert.Error(t, err, "non-TTY invocation without --yes should fail")
    assert.Contains(t, err.Error(), "non-interactive", "error should mention non-interactive mode")
}
```

Note: add `"fmt"` to the import block in the test file alongside `"strings"`, `"testing"`, `assert`, and `require`.

## Done When

- [ ] All requirements met
- [ ] `just build` and `BUILD_TAGS=dev just build` pass from the worktree root
- [ ] `just test unit` and `go test -tags dev ./...` pass
- [ ] `just lint` and `BUILD_TAGS=dev just lint` pass
- [ ] `just test integration` passes with the new `TestWorktreesClean_*` tests
- [ ] `camp worktrees clean --dry-run --all` in a real campaign prints the list but makes no changes
- [ ] `camp worktrees clean --all` in a non-TTY context without `--yes` exits non-zero

## Out of Scope

- GIT-8 worktree-vs-submodule detection improvements belong to sequence 11 (debt burn-down); this task adds guards on top of the existing detection without replacing it.
- `camp project worktree remove` (`cmd/camp/project/worktree/remove.go`) is a verified clean: it correctly routes through `git.Remove` and lets git refuse (line 90); do not modify it.
- The worktree `add` plumbing is verified clean by the second pass and must not be touched.