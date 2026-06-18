---
fest_type: task
fest_id: 04_git_plumbing_convergence.md
fest_name: git_plumbing_convergence
fest_parent: 11_debt_burn_down
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.787781-06:00
fest_updated: 2026-06-15T05:39:10.706965-06:00
fest_tracking: true
---


# Task: Git Plumbing Convergence and Porcelain Fixes

## Objective

Fix all correctness bugs in the git plumbing layer and converge the fragmented exec call sites onto a shared `RunGitCmd` primitive, covering GIT-4, GIT-5, GIT-8, ERR-8, and the documented follow-up note for the long tail.

## Requirements

- [ ] `internal/git/query.go:Output` no longer strips porcelain lines (GIT-4 porcelain fix; the `fmt.Errorf` migration is from task 01)
- [ ] `internal/git/commit.go FilterTracked` uses `-z` and NUL-split (GIT-5)
- [ ] `internal/git/errors.go ClassifyGitError` adds `LC_ALL=C` to git execs and tightens submodule match (ERR-8)
- [ ] `internal/git/submodule.go IsSubmodule` distinguishes worktrees from submodules by gitdir target (GIT-8)
- [ ] `internal/git/commit.go:327-331` exclusion pathspecs use `:(exclude,literal)` form (GIT-10)
- [ ] `internal/git/run.go:56` uses `ResolveGitDir` for the lock path instead of hardcoded `"index.lock"` (GIT-10)
- [ ] `internal/worktree/detect.go:125-128` relative gitdir joined against the worktree path, not CWD (GIT-10)
- [ ] `internal/git/resolve.go:80-82` uses `filepath.EvalSymlinks` in addition to `filepath.Abs` (GIT-10)
- [ ] `internal/worktree/paths.go:57` `HasPrefix` gains a separator guard (GIT-10)
- [ ] `cmd/camp/worktrees/commit.go:142` gains quest/workitem context via `PrependContextTagsFull` (GIT-10)
- [ ] Workitem-ref asymmetry fixed: `cmd/camp/project/commit_workitem.go` gains `EnsureRefForCommit` backfill (GIT-10)
- [ ] The `internal/git` package's own exec call sites converge onto `RunGitCmd` or `gitCmd`; a documented follow-up note covers the 51-file long tail
- [ ] Four named tests added: porcelain leading-space case, non-ASCII path tracked-filter, worktree-vs-submodule detection, locale-forced classification
- [ ] Both-profile gate green after every fix commit

## Implementation

### Background

GIT-4, GIT-5, GIT-8, GIT-10, and ERR-8 form a cluster of git plumbing correctness issues. The worktree is at `bc2ad1f`. Verify each file:line target before editing; some line numbers may have drifted.

The convergence onto `RunGitCmd` has two tiers: (1) the `internal/git` package itself converges fully in this task; (2) the 51 non-test files outside `internal/git` that call `exec.CommandContext("git", ...)` directly are too large a surface for one pass. A follow-up note in `docs/architecture/git-plumbing.md` records the remaining work.

### Step 1: GIT-4 - Fix porcelain TrimSpace in `internal/git/query.go`

Verified: `internal/git/query.go:16` returns `strings.TrimSpace(string(output))`. For porcelain v1 output like ` M file.go`, the leading-space XY status code is stripped, turning ` M` into `M`. The correct `-z` parser already exists at `internal/commands/workitem/staging_git.go:98-117` (`parseGitStatusPorcelainZ`).

Add to `internal/git/query.go` a dedicated porcelain helper that does NOT trim:

```go
// StatusPorcelain runs `git status --porcelain=v1 -z` and returns raw NUL-delimited output.
// Unlike Output, this preserves leading whitespace in XY status codes.
func StatusPorcelain(ctx context.Context, repoPath string) ([]byte, error) {
    fullArgs := []string{"-C", repoPath, "status", "--porcelain=v1", "-z"}
    cmd := exec.CommandContext(ctx, "git", fullArgs...)
    var stderr bytes.Buffer
    cmd.Stderr = &stderr
    output, err := cmd.Output()
    if err != nil {
        return nil, camperrors.Wrapf(err, "git status --porcelain=v1 -z: %s", stderr.String())
    }
    return output, nil
}
```

Update the status caller (`getRepoStatus` in `cmd/camp/status_all.go`, or `internal/status/status.go` after task 03 extraction) to call `StatusPorcelain` and parse via the `-z` parser. If `parseGitStatusPorcelainZ` is not importable from that location (import boundary), extract it to `internal/git/porcelain.go` where both can share it.

Add the test at `internal/git/porcelain_test.go`:
```go
func TestParseGitStatusPorcelainZLeadingSpace(t *testing.T) {
    // " M file.go" NUL-terminated: XY code " M", path "file.go"
    input := []byte(" M file.go\x00")
    entries := parseGitStatusPorcelainZ(input)
    if len(entries) != 1 || entries[0].Code != " M" || entries[0].Path != "file.go" {
        t.Fatalf("got %+v", entries)
    }
}
```

### Step 2: GIT-5 - Fix `FilterTracked` to use `-z`

Location: `internal/git/commit.go:200-243`. Replace `git ls-files` with `git ls-files -z` and NUL-split the output:

```go
func FilterTracked(ctx context.Context, repoPath string, paths []string) ([]string, error) {
    args := append([]string{"-C", repoPath, "ls-files", "-z", "--"}, paths...)
    cmd := exec.CommandContext(ctx, "git", args...)
    var stderr bytes.Buffer
    cmd.Stderr = &stderr
    output, err := cmd.Output()
    if err != nil {
        return nil, camperrors.Wrapf(err, "git ls-files -z: %s", stderr.String())
    }
    tracked := make(map[string]bool)
    for _, p := range bytes.Split(output, []byte{0}) {
        if len(p) > 0 {
            tracked[string(p)] = true
        }
    }
    var result []string
    for _, p := range paths {
        if tracked[p] {
            result = append(result, p)
        }
    }
    return result, nil
}
```

Test (table-driven, no real git required):
```go
func TestFilterTrackedNonASCII(t *testing.T) {
    // Feed mock output with a NUL-delimited list containing a UTF-8 path
    // Verify it is returned when the input paths list contains the same path
    // (The old C-quoted behavior would have dropped it)
}
```

### Step 3: GIT-8 - Fix `IsSubmodule` to distinguish worktrees

Location: `internal/git/submodule.go:17-30`. Current: returns true for any `.git`-as-a-file, which includes linked worktrees.

The fix reads the gitdir target and checks for `/.git/worktrees/`:

```go
func IsSubmodule(path string) (bool, error) {
    gitPath := filepath.Join(path, ".git")
    info, err := os.Stat(gitPath)
    if err != nil {
        if os.IsNotExist(err) {
            return false, nil
        }
        return false, camperrors.Wrapf(err, "cannot access .git at %s", path)
    }
    if info.IsDir() {
        return false, nil // regular repo
    }
    // .git is a file: submodule or worktree. Distinguish by gitdir target.
    gitdir, err := ResolveGitDir(path)
    if err != nil {
        return false, camperrors.Wrapf(err, "resolve gitdir at %s", path)
    }
    if strings.Contains(filepath.ToSlash(gitdir), "/.git/worktrees/") {
        return false, nil // linked worktree
    }
    return true, nil
}
```

Test with a t.TempDir() fixture (no real git, write the .git file content by hand):
```go
func TestIsSubmoduleDistinguishesWorktrees(t *testing.T) {
    cases := []struct {
        name     string
        gitfile  string
        expected bool
    }{
        {"submodule", "gitdir: ../../../.git/modules/foo\n", true},
        {"worktree", "gitdir: /abs/.git/worktrees/bar\n", false},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            dir := t.TempDir()
            if err := os.WriteFile(filepath.Join(dir, ".git"), []byte(tc.gitfile), 0644); err != nil {
                t.Fatal(err)
            }
            got, err := IsSubmodule(dir)
            if err != nil {
                t.Fatal(err)
            }
            if got != tc.expected {
                t.Errorf("IsSubmodule = %v, want %v", got, tc.expected)
            }
        })
    }
}
```

### Step 4: ERR-8 - LC_ALL=C and tighter submodule classification

Add a `gitCmd` helper in `internal/git/exec.go` (create the file):

```go
// gitCmd returns an exec.Cmd for git with LC_ALL=C to ensure predictable ASCII error messages
// regardless of the user's locale setting.
func gitCmd(ctx context.Context, args ...string) *exec.Cmd {
    cmd := exec.CommandContext(ctx, "git", args...)
    cmd.Env = append(os.Environ(), "LC_ALL=C", "LANG=C")
    return cmd
}
```

Converge `RunGitCmd` and `Output` to use `gitCmd` internally.

Tighten `ClassifyGitError`:

```go
// Add "no changes added to commit" as GitErrorNoChanges:
case strings.Contains(lower, "nothing to commit"),
    strings.Contains(lower, "no changes added to commit"):
    return GitErrorNoChanges

// Tighten submodule (was: any stderr containing "submodule"):
case strings.Contains(lower, "submodule update"),
    strings.Contains(lower, "no submodule mapping found"),
    strings.Contains(lower, "submodule '") :
    return GitErrorSubmodule
```

Test:
```go
func TestClassifyGitErrorLocale(t *testing.T) {
    cases := []struct{ stderr string; want GitErrorType }{
        {"nothing to commit, working tree clean", GitErrorNoChanges},
        {"no changes added to commit (use \"git add\"...)", GitErrorNoChanges},
        {"error: submodule update failed", GitErrorSubmodule},
    }
    for _, tc := range cases {
        got := ClassifyGitError(tc.stderr, 1)
        if got != tc.want {
            t.Errorf("ClassifyGitError(%q) = %v, want %v", tc.stderr, got, tc.want)
        }
    }
}
```

### Step 5: GIT-10 small fixes

Each is a targeted edit. Verify line numbers in the worktree before editing.

**`internal/git/commit.go:327-331` literal exclusion pathspecs:**
```go
// Before:
files = append(files, ":!"+p)
// After:
files = append(files, ":(exclude,literal)"+p)
```

**`internal/git/run.go:56` lock path:**
```go
// Before:
return &LockError{Path: "index.lock", Err: err}
// After: resolve against the repo path (repoPath must be in scope; check RunGitCmd signature)
lockPath := "index.lock"
if gitDir, resolveErr := ResolveGitDir(repoPath); resolveErr == nil {
    lockPath = filepath.Join(gitDir, "index.lock")
}
return &LockError{Path: lockPath, Err: err}
```

**`internal/worktree/detect.go:125-128`** - Read the actual lines first, then resolve relative gitdir against the worktree path rather than CWD:
```go
if !filepath.IsAbs(gitdir) {
    gitdir = filepath.Clean(filepath.Join(path, gitdir))
}
```

**`internal/git/resolve.go:80-82`** - Add EvalSymlinks:
```go
absRoot, _ := filepath.Abs(root)
if r, err := filepath.EvalSymlinks(absRoot); err == nil { absRoot = r }
absCamp, _ := filepath.Abs(campaignRoot)
if r, err := filepath.EvalSymlinks(absCamp); err == nil { absCamp = r }
if absRoot == absCamp {
```

**`internal/worktree/paths.go:57`** - Separator guard:
```go
// Before:
if !strings.HasPrefix(absPath, wtRoot) {
// After:
sep := string(filepath.Separator)
if !strings.HasPrefix(absPath, wtRoot+sep) && absPath != wtRoot {
```

**`cmd/camp/worktrees/commit.go:142`** - Replace `PrependCampaignTag` with `PrependContextTagsFull`:
Read the current line and its surrounding context. The function `resolveCommitContext` in `cmd/camp/commit_workitem.go` is the pattern. After task 03 moves `EnsureRefForCommit` to `internal/workitem`, use:
```go
questID, wiRef := resolveWorktreeCommitContext(ctx, campRoot)
message = git.PrependContextTagsFull(cfg.ID, questID, wiRef, message)
```
where `resolveWorktreeCommitContext` is a small local helper following the same pattern as `resolveCommitContext`.

**`cmd/camp/project/commit_workitem.go`** - Add EnsureRefForCommit backfill:
After task 03 moves `EnsureRefForCommit` to `internal/workitem`, update `resolveProjectCommitContext` to call it the same way root commit does.

### Step 6: Converge `internal/git` onto RunGitCmd

Read each hand-rolled exec block in `internal/git/commit.go` and `internal/worktree/git.go`, then replace with `RunGitCmd` or `gitCmd`. The streaming variant (for commands that pipe stdout) is a follow-up. For `pkg/commitkit/commitkit.go:169` (`ShortHash`):
```go
out, err := git.Output(ctx, root, "rev-parse", "--short", "HEAD")
```

Create `docs/architecture/git-plumbing.md` with one paragraph documenting the 51-file long tail.

## Done When

- [ ] `TestParseGitStatusPorcelainZLeadingSpace` passes (leading-space XY code preserved)
- [ ] `TestFilterTrackedNonASCII` passes (UTF-8 path included in FilterTracked result)
- [ ] `TestIsSubmoduleDistinguishesWorktrees` passes (`.git` pointing at `.git/worktrees/` returns false)
- [ ] `TestClassifyGitErrorLocale` passes including `"no changes added to commit"` as `GitErrorNoChanges`
- [ ] `grep -n '":!"' internal/git/commit.go` returns 0 results
- [ ] `grep -n '"index.lock"' internal/git/run.go` returns 0 results
- [ ] `docs/architecture/git-plumbing.md` exists with the 51-file follow-up note
- [ ] Both-profile gate green after each commit