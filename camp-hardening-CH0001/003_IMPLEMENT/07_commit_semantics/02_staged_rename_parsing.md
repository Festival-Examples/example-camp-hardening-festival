---
fest_type: task
fest_id: 02_staged_rename_parsing.md
fest_name: staged_rename_parsing
fest_parent: 07_commit_semantics
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.713881-06:00
fest_tracking: true
---

# Task: Fix listStagedFiles to include both rename sides (N-5)

## Objective

Replace `diff --cached --name-only -z` in `listStagedFiles` with `diff --cached --name-status -z` parsed for both source and destination of R/C records, so that `camp workitem commit --staged` on a staged rename commits the full rename without leaving a stray staged deletion.

## Requirements

- [x] `listStagedFiles` at `internal/commands/workitem/staging_git.go:55-61` switches from `--name-only` to `--name-status` and parses both paths of R/C (rename/copy) status records.
- [x] The new parser includes the source path (the delete side) so the commit plan carries it and no stray staged `D` remains after `--staged` on a rename.
- [x] The parsing logic reuses or mirrors the existing `-z` parser in `ExpandTrackedPaths` (`internal/git/commit.go:250-306`) to avoid drift between the two implementations.
- [x] `assertCleanIndex` at `staging_git.go:18-33` calls the updated `listStagedFiles`; its behavior is unchanged (it still lists all staged files and refuses if any are present before an auto-commit); the fix just ensures renames are counted correctly on both sides.
- [x] Regression test: staged `git mv a.txt b.txt` followed by `workitem commit --staged` (or the underlying `stageAndCommit` with `PreStaged` carrying both paths) results in a clean index with `a.txt` absent from HEAD and `b.txt` present, and `git diff --cached` is empty afterward.

## Implementation

### Background: why --name-only drops the delete side

`git diff --cached --name-only` for a staged rename `a.txt -> b.txt` emits only `b.txt` (the destination). The source deletion is implicit in the rename record but is not printed by `--name-only`. When `listStagedFiles` returns only `b.txt`, the commit plan for `--staged` feeds `["b.txt"]` to `stageAndCommit` as `PreStaged`. This path (after task 01's fix) copies the real index and commits from it -- but the SCOPE assertion only checks `b.txt`, so if `a.txt`'s deletion is not in `PreStaged`, the subsequent `assertCleanIndex` call (which runs AFTER the commit) will see the stray `D a.txt` and refuse future commits.

The full chain of harm (reproduced by the review's git pass):

```
git mv a.txt b.txt            # stages: D a.txt, A b.txt (rename pair)
listStagedFiles returns: [b.txt]   # drop the D side
workitem commit --staged
  stageAndCommit(PreStaged=[b.txt])
  -> copies real index to temp (has both D a.txt and A b.txt)
  -> ExpandTrackedPathsFromTempIndex([b.txt])
     -> returns [b.txt]  (only the add side is in scope)
  -> commit creates a commit with only b.txt added (A b.txt)
  -> D a.txt stays staged in the REAL index (not committed!)
next workitem commit --staged
  assertCleanIndex fires: "1 pre-existing staged file: a.txt"
```

With the corrected `listStagedFiles` returning `["a.txt", "b.txt"]`, the `PreStaged` slice carries both paths, `ExpandTrackedPathsFromTempIndex` includes both, and the commit is a true rename commit.

### Verified file:line targets (worktree HEAD bc2ad1f)

- `internal/commands/workitem/staging_git.go:55-61` -- `listStagedFiles` function; `diff --cached --name-only -z` at :56; `parseNULPathList` at :60.
- `internal/commands/workitem/staging_git.go:98-117` -- `parseGitStatusPorcelainZ`; this is the correct parser for R/C two-path records referenced by R31; note the `R`/`C` branch at :112-114 which skips the old-name field.
- `internal/git/commit.go:250-306` -- `ExpandTrackedPaths`; the authoritative `-z` name-status parser; handles R/C at :289-297.
- `internal/commands/workitem/staging_git.go:18-33` -- `assertCleanIndex`; calls `listStagedFiles` at :19.

Note: `parseGitStatusPorcelainZ` at :98-117 parses `git status --porcelain -z` format, where each entry is a two-character code followed by a space then the path, and R/C entries have the old path as a second NUL-separated field. `diff --cached --name-status -z` uses a DIFFERENT format: each entry is a status letter (e.g. `R100`) NUL, then destination path NUL, then source path NUL (for rename records). Do NOT reuse `parseGitStatusPorcelainZ` for the diff output; reuse or extract the parser from `ExpandTrackedPaths` instead.

### Step 1: Extract the -z name-status parser from ExpandTrackedPaths

In `internal/git/commit.go`, extract the body of the NUL-field loop (lines ~:268-304) into an unexported helper:

```go
// parseNameStatusZ parses `git diff --name-status -z` output.
// For R/C records it returns both the destination and source paths.
// For all other records it returns the single path.
func parseNameStatusZ(output []byte) ([]string, error) {
    if len(output) == 0 {
        return nil, nil
    }
    fields := strings.Split(string(output), "\x00")
    seen := make(map[string]struct{}, len(fields))
    result := make([]string, 0, len(fields))
    addPath := func(path string) {
        if path == "" {
            return
        }
        if _, ok := seen[path]; ok {
            return
        }
        seen[path] = struct{}{}
        result = append(result, path)
    }
    for i := 0; i < len(fields); {
        status := fields[i]
        i++
        if status == "" {
            continue
        }
        switch status[0] {
        case 'R', 'C':
            if i+1 >= len(fields) {
                return nil, camperrors.NewGit("diff --cached", "", "", "malformed rename/copy status output", nil)
            }
            addPath(fields[i])
            addPath(fields[i+1])
            i += 2
        default:
            if i >= len(fields) {
                return nil, camperrors.NewGit("diff --cached", "", "", "malformed diff status output", nil)
            }
            addPath(fields[i])
            i++
        }
    }
    return result, nil
}
```

Update `ExpandTrackedPaths` to call `parseNameStatusZ` instead of duplicating the loop. This is a refactor-only change to that function; its behavior is unchanged (C-3).

### Step 2: Update listStagedFiles

Replace the existing `listStagedFiles` body:

```go
// Before (staging_git.go:55-61):
func listStagedFiles(ctx context.Context, repoRoot string) ([]string, error) {
    out, err := exec.CommandContext(ctx, "git", "-C", repoRoot, "diff", "--cached", "--name-only", "-z").Output()
    if err != nil {
        return nil, err
    }
    return parseNULPathList(out), nil
}

// After:
func listStagedFiles(ctx context.Context, repoRoot string) ([]string, error) {
    out, err := exec.CommandContext(ctx, "git", "-C", repoRoot, "diff", "--cached", "--name-status", "-z").Output()
    if err != nil {
        return nil, camperrors.NewGit("diff --cached --name-status", "", "", "", err)
    }
    // git is the import alias for internal/git; parseNameStatusZ is exported from
    // internal/git/commit.go (or unexported and accessed via a thin exported wrapper).
    // If the helper is package-private, duplicate the logic here or move it to
    // an internal/git package function.
    return git.ParseNameStatusZ(out)
}
```

If `parseNameStatusZ` is unexported in `internal/git`, export it as `ParseNameStatusZ` or add a thin public wrapper `ParseDiffNameStatusZ(output []byte) ([]string, error)` in `internal/git/commit.go`. The `staging_git.go` file already imports `internal/git` indirectly via the `camperrors` alias; confirm the import path and add it if not present.

### Step 3: Verify assertCleanIndex behavior

`assertCleanIndex` at :18-33 calls `listStagedFiles` and refuses if any paths are returned. After the fix, a staged `git mv a.txt b.txt` will report `["a.txt", "b.txt"]`, causing `assertCleanIndex` to correctly block an unrelated auto-commit. This is the desired behavior: the user must either complete the rename commit or unstage before running an auto-commit.

The hint message at :30-32 should continue to reference both the `--staged` path and `git reset HEAD` as recovery options. No change needed to the message text.

### Step 4: Verify the commit plan carries both paths for --staged

After task 01's engine is in place, verify that `stageAndCommit` called with `PreStaged: ["a.txt", "b.txt"]` (as `listStagedFiles` will now return) results in:

1. A temp index that is a copy of the real index (Mode B path).
2. `ExpandTrackedPathsFromTempIndex(["a.txt", "b.txt"])` returns both paths.
3. The commit carries the rename: `a.txt` is deleted from HEAD, `b.txt` is added.
4. The real index is clean after the commit (no stray staged entries).

This is the sequence end-to-end test; write it as a regression test.

### Regression test

Add a test in `internal/commands/workitem/staging_git_test.go` (or `commit_test.go` in the same package, whichever exists):

**Test: staged rename commits cleanly**

```
scratch repo:
  echo "content" > a.txt
  git add a.txt
  git commit -m "init"
  git mv a.txt b.txt         # staged rename; a.txt deleted, b.txt added

call listStagedFiles(ctx, repoRoot)
assert result contains both "a.txt" and "b.txt"

call stageAndCommit with opts.PreStaged = listStagedFiles result, opts.Files = nil
assert no error
assert HEAD:b.txt exists with "content\n"
assert HEAD:a.txt does not exist
assert git diff --cached is empty (assertCleanIndex would pass)
```

If `stageAndCommit` is not directly accessible from the workitem package's test, write the test in `internal/git/commit/commit_test.go` and test `listStagedFiles` separately as a unit test in the workitem package.

## Done When

- [x] All requirements met
- [x] `listStagedFiles` uses `--name-status -z` and returns both sides of rename/copy records
- [x] `parseNameStatusZ` (or equivalent) is a shared helper; `ExpandTrackedPaths` calls it without duplicating the loop
- [x] Regression test passes: staged `git mv a.txt b.txt` then `--staged` commit leaves a clean index
- [x] `just build` and `BUILD_TAGS=dev just build` pass
- [x] `just test unit` (or `go test ./internal/commands/workitem/... ./internal/git/...`) passes with the new test
- [x] `just lint` (stable and dev) passes

## Completion Notes

- Project commit: `e2488b8` (`fix: include rename sources in staged commits`).
- `listStagedFiles` now runs `git diff --cached --name-status -z` and parses it through the shared `internal/git.ParseDiffNameStatusZ` wrapper.
- `ExpandTrackedPaths` and workitem staging now share the same `parseNameStatusZ` logic for R/C records.
- `assertCleanIndex` behavior remains unchanged, but staged renames are counted on both sides.
- Existing task-01 commit-engine regression coverage verifies staged rename commits leave a clean index with `a.txt` absent from `HEAD`, `b.txt` present, and no cached diff. Task 02 added pure parser coverage without adding new host-side git test violations.
- Verification passed: `go test -count=1 -short ./internal/commands/workitem ./internal/git/... ./internal/git/commit`, `go test -tags dev -count=1 -short ./internal/commands/workitem ./internal/git/... ./internal/git/commit`, `just lint`, `BUILD_TAGS=dev just lint`, `just build stable`, and `just build dev`.

## Out of Scope

- `ExpandTrackedPaths` behavior change: the refactor to call `parseNameStatusZ` must produce identical output for all inputs; if any discrepancy is found during refactoring, stop and record it as a bug rather than silently changing behavior (C-3).
- `parseGitStatusPorcelainZ`: do not change; it parses `git status --porcelain`, which has different field layout from `diff --name-status`. Confusion between the two formats is the root cause of N-5.
