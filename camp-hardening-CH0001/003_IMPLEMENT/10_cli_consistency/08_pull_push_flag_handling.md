---
fest_type: task
fest_id: 08_pull_push_flag_handling.md
fest_name: pull_push_flag_handling
fest_parent: 10_cli_consistency
fest_order: 8
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.771142-06:00
fest_updated: 2026-06-15T03:15:18.510611-06:00
fest_tracking: true
---


# Task: Pull/Push Flag Handling Fix

## Objective

Stop `ExtractSubFlags` from hijacking git's bare `-p` shorthand (which git uses for `--prune` in `git pull` and `--patch` in `git push`), honor the `--` argument terminator, and fix the error hint in `handlePullError` to use a resolvable `projects/<path>` form instead of the display name that `resolveProjectPath` cannot match.

## Requirements

- [ ] `internal/git/resolve.go:117-144` (`ExtractSubFlags`): the `case arg == "--project" || arg == "-p":` branch is changed to intercept only `--project` and `--project=...` forms. The bare `-p` case is removed. If the caller needs a short form for `--project`, they must use `--project=<path>` or `--project <path>`.
- [ ] `internal/git/resolve.go:ExtractSubFlags`: add `--` terminator handling. When `--` is encountered, all subsequent arguments are added to `remaining` without inspection.
- [ ] `cmd/camp/pull.go:312`: the error hint `camp pull -p %s --no-rebase` (using `t.name`, a display name) is corrected to `camp pull --project=projects/%s --no-rebase` (using the relative path `t.relPath` or an equivalent field; see the edge cases section about how to get the path).
- [ ] A test in `internal/git/resolve_test.go` (host `t.TempDir()` convention, or an existing test file in that package) verifies:
  - `camp pull -p origin` passes `-p origin` to git untouched (i.e., `remaining` contains `-p` and `origin`).
  - `camp pull --project=projects/camp origin` extracts `project="projects/camp"` and `remaining=["origin"]`.
  - `camp pull --project projects/camp origin` extracts `project="projects/camp"` and `remaining=["origin"]`.
  - `camp pull -- -p origin` passes both `-p` and `origin` to git (terminator respected).
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

Finding N-13 from the second pass: `internal/git/resolve.go:127-132` in `ExtractSubFlags` intercepts bare `-p` as `--project`. This means:

- `camp pull -p origin` (user means git `--prune`): camp consumes `-p` and treats `origin` as a project path, stripping both from the git args. Git receives no remote and no prune flag, and the pull silently targets the wrong repo.
- `camp pull -p origin` (user means git push `--patch`): same problem for push.
- There is no `--` terminator handling, so users cannot escape camp's flag scanning with the standard shell convention.

The `cmd/camp/pull.go:312` hint `camp pull -p %s --no-rebase` uses `t.name` which is the display name (e.g., `"camp"` from `git.SubmoduleDisplayName(p)`). The `resolveProjectPath` function at the start of pull's RunE (around line 62) uses `git.ExtractSubFlags` which accepts a project path, not a display name. A user who copies the hint and runs it will not find the project by display name.

Both commands use `DisableFlagParsing: true` (verified at `cmd/camp/pull.go:45` and `cmd/camp/push.go:41`), which is why `ExtractSubFlags` exists at all. The fix to `ExtractSubFlags` is in the shared `internal/git/resolve.go`, so it applies to both `pull` and `push` automatically.

Verify line numbers against the worktree at `bc2ad1f` before editing.

### Verified file:line targets (as of review citations; verify before editing)

- `internal/git/resolve.go:117-144` - `ExtractSubFlags` function body
- `cmd/camp/pull.go:312` - error hint with `t.name`
- `cmd/camp/pull.go:143-148` - `pullTarget` struct definition
- `cmd/camp/pull.go:346-368` - `buildPullTargets` constructing the struct

### Step-by-step approach

**Step 1: Fix `ExtractSubFlags` to not intercept bare `-p`.**

Read `internal/git/resolve.go`. Find `ExtractSubFlags`. The current switch body contains:

```go
case arg == "--project" || arg == "-p":
    // Next arg is the project path
    if i+1 < len(args) {
        i++
        project = args[i]
    }
```

Change to:

```go
case arg == "--project":
    // Next arg is the project path
    if i+1 < len(args) {
        i++
        project = args[i]
    }
```

Remove the `|| arg == "-p"` condition. The `--project=...` form is handled by the existing `strings.HasPrefix(arg, "--project=")` case; no change needed there.

**Step 2: Add `--` terminator handling.**

In the same `for` loop, add a case at the top of the switch (before any other cases) or as an early exit:

```go
case arg == "--":
    // Everything after -- is passed to git unchanged.
    remaining = append(remaining, args[i:]...)
    return remaining, sub, project
```

This causes `ExtractSubFlags` to stop scanning at `--` and append all remaining arguments (including `--` itself) to `remaining`, so git receives them verbatim.

**Step 3: Fix the pull error hint.**

Read `cmd/camp/pull.go`. Examine the `pullTarget` struct near line 143:

```go
type pullTarget struct {
    name   string
    path   string
    branch string
    isRoot bool
}
```

The `path` field is the absolute path to the project on disk (e.g., `<campaign-root>/projects/camp`). To construct the hint, the relative path from the campaign root is needed. The display name `t.name` comes from `git.SubmoduleDisplayName(p)` where `p` is the relative path. So the relative path is already available at construction time in `buildPullTargets`.

To fix the hint without changing the existing struct fields, add a `relPath string` field to `pullTarget` that stores the relative path (the `p` variable in `buildPullTargets`):

```go
type pullTarget struct {
    name    string
    path    string
    relPath string // relative path from campaign root, e.g. "projects/camp"
    branch  string
    isRoot  bool
}
```

In `buildPullTargets`:

```go
targets = append(targets, pullTarget{
    name:    git.SubmoduleDisplayName(p),
    path:    fullPath,
    relPath: p,
    branch:  branch,
})
```

Then in `handlePullError` at line 312, change the hint:

```go
errMsg: fmt.Sprintf("  %s: rebase conflict (try: camp pull --project=%s --no-rebase)", t.name, t.relPath),
```

This produces a resolvable form like `camp pull --project=projects/camp --no-rebase`. The user can copy and run it. The `resolveProjectPath` function accepts relative paths from the campaign root, so `projects/camp` will resolve correctly.

**Step 4: Write tests for `ExtractSubFlags`.**

In `internal/git/resolve_test.go` (or a new `extract_subflags_test.go` in the same package), write table-driven tests:

```go
func TestExtractSubFlags(t *testing.T) {
    tests := []struct {
        name            string
        args            []string
        wantRemaining   []string
        wantSub         bool
        wantProject     string
    }{
        {
            name:          "bare -p passes through to git untouched",
            args:          []string{"-p", "origin"},
            wantRemaining: []string{"-p", "origin"},
            wantSub:       false,
            wantProject:   "",
        },
        {
            name:          "--project= form extracts project",
            args:          []string{"--project=projects/camp", "origin"},
            wantRemaining: []string{"origin"},
            wantProject:   "projects/camp",
        },
        {
            name:          "--project space form extracts project",
            args:          []string{"--project", "projects/camp", "origin"},
            wantRemaining: []string{"origin"},
            wantProject:   "projects/camp",
        },
        {
            name:          "-- terminator stops flag extraction",
            args:          []string{"--", "-p", "origin"},
            wantRemaining: []string{"--", "-p", "origin"},
        },
        {
            name:          "--sub extracted",
            args:          []string{"--sub", "origin"},
            wantRemaining: []string{"origin"},
            wantSub:       true,
        },
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            remaining, sub, project := ExtractSubFlags(tt.args)
            if !slices.Equal(remaining, tt.wantRemaining) {
                t.Errorf("remaining: got %v, want %v", remaining, tt.wantRemaining)
            }
            if sub != tt.wantSub {
                t.Errorf("sub: got %v, want %v", sub, tt.wantSub)
            }
            if project != tt.wantProject {
                t.Errorf("project: got %q, want %q", project, tt.wantProject)
            }
        })
    }
}
```

Use `slices.Equal` from `slices` package (Go 1.21+) or write a simple equality helper.

### Edge cases

**Campaign root target has no `relPath`:** The campaign root target in `buildPullTargets` is constructed with `name: "campaign root"` and no equivalent relative path. It should have `relPath: ""` or `relPath: "."`. In `handlePullError`, the campaign root entry would have `t.isRoot == true`; the rebase hint is only relevant for submodule entries, so this case is fine.

**`-p` as shorthand for `--project`:** The old behavior allowed `-p <path>` as a shorthand for `--project <path>`. Users who relied on this will need to use `--project` instead. Since camp is not publicly released, this is acceptable. The docs recipe in task 05 pins stable docs; update the pull command Long help or Example field to show `--project` explicitly and not mention `-p` as a project shorthand.

**`push.go` also uses `ExtractSubFlags`:** The fix automatically applies to both `pull` and `push` because `ExtractSubFlags` is in shared code. Verify that `cmd/camp/push.go` does not have any `-p` usage in its help text or examples that would become misleading. Read `cmd/camp/push.go` quickly to confirm. The finding only mentions a hint fix in `pull.go`; push has a similar hint at `cmd/camp/push.go:227` that was verified accurate ("push") so it does not need to change.

### Out of scope

- `cmd/camp/push.go` hint text at line 227 was verified accurate in the findings ("Verified accurate: push.go wording") and is in the C-3 verified-negatives list. Do not touch it.
- Full `--project` shorthand `-p` with proper cobra flag registration (i.e., replacing `DisableFlagParsing` with real cobra flags for these commands) is a larger refactor in the ARCH-2 debt area; leave it for sequence 11 or 12.

## Done When

- [ ] All requirements met
- [ ] `ExtractSubFlags([]string{"-p", "origin"})` returns `remaining=[-p origin]`, `project=""`
- [ ] `ExtractSubFlags([]string{"--", "-p", "origin"})` returns `remaining=[-- -p origin]` (terminator respected)
- [ ] `ExtractSubFlags([]string{"--project=projects/camp", "origin"})` returns `remaining=[origin]`, `project="projects/camp"`
- [ ] All four table-driven tests in `TestExtractSubFlags` pass in both profiles
- [ ] `handlePullError` hint uses `--project=<relPath>` form
- [ ] Both-profile gate passes