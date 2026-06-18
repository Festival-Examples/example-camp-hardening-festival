---
fest_type: task
fest_id: 03_pull_all_rebase_guard.md
fest_name: pull_all_rebase_guard
fest_parent: 03_sync_and_migration_safety
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.637886-06:00
fest_updated: 2026-06-14T19:55:34.249429-06:00
fest_tracking: true
---


# Task: Prevent camp pull all from aborting pre-existing rebases

**Finding:** R7 / GIT-3

## Objective

Probe `IsRebaseInProgress` BEFORE attempting to pull each target in `runPullAll`; skip repos that were already mid-rebase with a warning so that `handlePullError` never aborts a rebase the pull loop did not start.

## Requirements

- [ ] `pullSingleTarget` probes `git.IsRebaseInProgress(ctx, t.path)` before calling `runGitPullWithLockRetry`; if a rebase is already in progress, the repo is skipped with a printed warning and `pullOutcomeSkipped` is returned.
- [ ] `handlePullError` is modified so that `abortRebase` is called only when the pull loop itself initiated the rebase (tracked via a boolean passed to or returned from the pull attempt).
- [ ] The single-repo `camp pull` path (`cmd/camp/pull.go:69-88`) is not changed; it already prints guidance and returns the error without aborting.
- [ ] Container test: a repo with a pre-existing conflicted rebase is included in a multi-repo `camp pull all` run; after the run the conflicted rebase state is intact (REBASE_HEAD exists, `git status` shows rebase in progress).
- [ ] The hostile-remote fixture from task 01 can serve as the other repo in the multi-repo scenario to cover both a skipped-rebase repo and a normal pull in one test run (optional but preferred for fixture reuse).

## Implementation

### Background

`handlePullError` at `cmd/camp/pull.go:305-314` (worktree bc2ad1f, verified at `:306-314`):

```go
func handlePullError(ctx context.Context, t *pullTarget, output []byte, err error, red lipgloss.Style) pullResult {
    if git.IsRebaseInProgress(ctx, t.path) {
        _ = abortRebase(ctx, t.path)
        fmt.Println(red.Render("conflict (aborted rebase)"))
        return pullResult{
            outcome: pullOutcomeFailed,
            errMsg:  fmt.Sprintf("  %s: rebase conflict (try: camp pull -p %s --no-rebase)", t.name, t.name),
        }
    }
```

This runs AFTER the pull has failed. There is no check whether the rebase existed before the pull started. Any repo mid-conflict-resolution before `camp pull all` will have its progress destroyed.

The single-repo path at `:77-84` correctly prints guidance without aborting. The fix makes the multi-repo path match that behavior for pre-existing rebases.

### Step 1: Add a pre-pull rebase check in pullSingleTarget

In `pullSingleTarget` (`cmd/camp/pull.go:231`), after the detached-HEAD and upstream-tracking checks and before the `runGitPullWithLockRetry` call, insert:

```go
// Skip repos already mid-rebase; only abort rebases this loop initiates.
if git.IsRebaseInProgress(ctx, t.path) {
    fmt.Printf("  %-30s %s\n", t.name,
        yellow.Render("skipped (rebase in progress -- resolve or abort manually)"))
    return pullResult{outcome: pullOutcomeSkipped}
}
```

The exact insertion point is after the upstream-tracking block (around line 273) and before the "Print progress line" comment block (around line 275).

### Step 2: Track loop-initiated rebases and guard handlePullError

After the pre-pull check fires for pre-existing rebases, the only remaining path to `handlePullError` with a rebase in progress is one that the pull itself triggered (e.g. `git pull --rebase` started a rebase and hit a conflict). Track this explicitly.

In `pullSingleTarget`, after calling `runGitPullWithLockRetry` and before passing the error to `handlePullError`, check whether the rebase appeared as a result of the pull:

```go
output, err := runGitPullWithLockRetry(ctx, t.path, pullArgs, false)
if err != nil {
    rebaseInitiatedHere := git.IsRebaseInProgress(ctx, t.path)
    return handlePullError(ctx, t, output, err, red, rebaseInitiatedHere)
}
```

Modify `handlePullError` to accept and use the new parameter:

```go
func handlePullError(ctx context.Context, t *pullTarget, output []byte, err error, red lipgloss.Style, rebaseInitiatedHere bool) pullResult {
    if git.IsRebaseInProgress(ctx, t.path) {
        if rebaseInitiatedHere {
            _ = abortRebase(ctx, t.path)
            fmt.Println(red.Render("conflict (aborted rebase)"))
            return pullResult{
                outcome: pullOutcomeFailed,
                errMsg:  fmt.Sprintf("  %s: rebase conflict (try: camp pull -p %s --no-rebase)", t.name, t.name),
            }
        }
        // Pre-existing rebase reached handlePullError despite the pre-pull guard.
        // Do not abort. This is a defensive fallback.
        fmt.Println(red.Render("failed (pre-existing rebase in progress; not aborted)"))
        return pullResult{
            outcome: pullOutcomeFailed,
            errMsg:  fmt.Sprintf("  %s: pull failed; rebase in progress -- resolve it manually", t.name),
        }
    }
    fmt.Println(red.Render("failed"))
    errMsg := strings.TrimSpace(string(output))
    if isDivergentError(errMsg) {
        errMsg = "branches diverged (try: camp pull all --ff-only, --rebase, or resolve manually)"
    } else if errMsg == "" {
        errMsg = err.Error()
    }
    return pullResult{
        outcome: pullOutcomeFailed,
        errMsg:  fmt.Sprintf("  %s: %s", t.name, errMsg),
    }
}
```

### Step 3: Container test

Place in `tests/integration/pull_rebase_guard_test.go` with a `//go:build integration` tag.

Test setup:
1. Create a bare origin with commits A and B.
2. Clone it as repo-1 (a normal submodule that will be pulled successfully).
3. Clone it as repo-2 and start a conflicting rebase: create a local commit C, then `git rebase origin/main` which conflicts; leave the conflict unresolved.
4. Add both repos as submodules to a campaign-root bare clone and `camp init` it.
5. Run `camp pull all` against the campaign root.
6. Assert: repo-1 reflects the latest upstream state.
7. Assert: repo-2 was skipped (the output mentions "rebase in progress").
8. Assert: `REBASE_HEAD` still exists in repo-2's git directory.
9. Assert: `git status` in repo-2 still shows the unresolved conflict (not "nothing to commit").

This task lives in `cmd/camp/pull.go` which is cmd/ logic. Sequence 11.03 will extract it into `internal/`; keep the fix self-contained here.

### Edge Cases

- **`--rebase` flag passed to pull all**: The pre-existing check fires before the pull attempt regardless of flags. The user must resolve the existing rebase first.
- **TOCTOU between pre-pull check and git pull**: Narrow window; the race causes the same behavior as today (pull fails, handlePullError handles it with the step 2 guard ensuring no abort on pre-existing rebases).
- **Multiple repos with pre-existing rebases**: Each is skipped independently; the loop continues to the remaining repos.

## Out of Scope

- Extracting pull logic to `internal/` (sequence 11.03).
- Fixing the hint text `camp pull -p <display-name>` (N-13, sequence 10).

## Done When

- [ ] `pullSingleTarget` probes `IsRebaseInProgress` before pulling; pre-existing rebases skip with a warning
- [ ] `handlePullError` accepts `rebaseInitiatedHere bool`; aborts only self-initiated rebases
- [ ] Container test passes: pre-existing conflicted rebase state is intact after `camp pull all`
- [ ] Single-repo `camp pull` path (`:69-88`) is unchanged
- [ ] Both-profile gate passes after these changes