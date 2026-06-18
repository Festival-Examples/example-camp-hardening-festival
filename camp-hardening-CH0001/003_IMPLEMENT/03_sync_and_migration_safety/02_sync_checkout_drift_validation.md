---
fest_type: task
fest_id: 02_sync_checkout_drift_validation.md
fest_name: sync_checkout_drift_validation
fest_parent: 03_sync_and_migration_safety
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.637649-06:00
fest_updated: 2026-06-14T19:45:32.28621-06:00
fest_tracking: true
---


# Task: Surface sync checkout errors and fix validateUpdate's backwards drift logic

**Finding:** N-10

## Objective

Make `camp sync` surface per-submodule checkout failures and correctly flag drift instead of silently excusing it, so a machine with a stale local `main` produces a loud non-success result rather than a false "everything is correct" exit.

## Requirements

- [ ] The `CheckoutDefaultBranch` call in `Syncer.initSubmodules` (worktree `internal/sync/sync.go:250`) captures both return values; a checkout error is recorded in the submodule result and printed, not silently discarded.
- [ ] `validateUpdate` (`:384-431`) is corrected: `+` lines in `git submodule status` output indicate that the checked-out commit differs from the recorded gitlink, which is a drift condition, not an expected post-sync state. The comment claiming otherwise is removed.
- [ ] When a submodule's local branch tip does not match the recorded gitlink after sync, `camp sync` either fast-forwards the local branch to the recorded commit (if the commit is an ancestor) or emits a clear per-submodule warning naming both the expected SHA and the actual branch tip.
- [ ] Sync exit code reflects drift warnings: if `--strict` (or the sync options already in the table) is set, drift is an error; otherwise it is a warning printed to stderr. The specific option name depends on the existing options table in `sync.go`.
- [ ] Unit tests: at least one table-driven test for `validateUpdate` exercising the `-` (not-initialized), `+` (drift), and ` ` (correct) prefix cases, asserting that `+` returns an error or warning rather than being silently skipped.
- [ ] Container test: a scenario where the recorded gitlink points to a commit that exists but the local branch tip is one commit behind; sync exits with a non-success report; a subsequent `camp refs sync` does not regress the recorded pointer to the stale local tip.

## Implementation

### Background

In `internal/sync/sync.go`, `initSubmodules` (around line 220) calls `git.CheckoutDefaultBranch` on line 250 but discards the return:

```go
// line 250 (worktree bc2ad1f, verified)
git.CheckoutDefaultBranch(ctx, subDir)
```

`CheckoutDefaultBranch` returns `(string, error)`. Both values are thrown away. If the checkout fails (e.g. untracked files would be overwritten, or the branch does not exist locally), the submodule result still reports success.

`validateUpdate` at line 384 reads `git submodule status --recursive` output. A `+` prefix means the working-tree commit differs from the gitlink recorded in the parent index. The current code has:

```go
// '+' prefix means commit differs, but this is expected after sync
// since we just updated to the recorded commit
```

This comment is backwards. `git submodule update` puts the submodule at the RECORDED commit in detached HEAD. If we then `CheckoutDefaultBranch` and the local branch tip is NOT the recorded commit, the `+` is real drift. Silently skipping it means `camp sync` exits 0 while the repo's working tree is sitting at a different commit than the gitlink says.

### Step 1: Capture the CheckoutDefaultBranch result

In `initSubmodules`, replace line 250:

```go
// Before:
git.CheckoutDefaultBranch(ctx, subDir)

// After:
if branch, checkoutErr := git.CheckoutDefaultBranch(ctx, subDir); checkoutErr != nil {
    result.Error = &SyncError{
        Op:        "checkout",
        Submodule: path,
        Cause:     checkoutErr,
    }
    result.Success = false
} else {
    result.CheckedOutBranch = branch
}
```

Add `CheckedOutBranch string` to the `SubmoduleResult` struct if it is not already present.

### Step 2: Correct validateUpdate

Replace the `+` handling block in `validateUpdate` (the comment and its silent skip):

```go
if strings.HasPrefix(line, "+") {
    parts := strings.Fields(line)
    sub := line
    recordedSHA := ""
    if len(parts) >= 2 {
        sub = parts[1]
        recordedSHA = strings.TrimPrefix(parts[0], "+")
    }
    // '+' means the working-tree commit differs from the recorded gitlink.
    // This is drift, not an expected post-sync state.
    return &SyncError{
        Op:        "validate-drift",
        Submodule: sub,
        Cause: camperrors.Newf(
            "%s: checked-out commit (%s) differs from recorded gitlink; run 'camp pull' to fast-forward or investigate",
            sub, recordedSHA,
        ),
    }
}
```

If the caller of `validateUpdate` should treat drift as a warning rather than a hard error in non-strict mode, the function can return a `[]DriftWarning` alongside the error; the exact shape follows the existing `SyncError` pattern in the file. The key invariant is that `+` drift is NEVER silently discarded.

### Step 3: Fast-forward on drift where possible

After `initSubmodules` returns results, add a per-submodule drift check. For each submodule where `result.CheckedOutBranch != ""`:

```go
recordedSHA, shaErr := git.RevParse(ctx, s.repoRoot, path)
branchSHA, tipErr := git.Output(ctx, subDir, "rev-parse", result.CheckedOutBranch)
if shaErr == nil && tipErr == nil &&
    strings.TrimSpace(branchSHA) != strings.TrimSpace(recordedSHA) {
    // Attempt fast-forward to the recorded commit.
    ffErr := exec.CommandContext(ctx, "git", "-C", subDir,
        "merge", "--ff-only", recordedSHA).Run()
    if ffErr != nil {
        // Cannot fast-forward; record the drift warning.
        result.DriftWarning = fmt.Sprintf(
            "%s: local %s tip %s != recorded %s (fast-forward failed)",
            path, result.CheckedOutBranch,
            strings.TrimSpace(branchSHA)[:8],
            strings.TrimSpace(recordedSHA)[:8],
        )
    }
}
```

`git.RevParse` reads the recorded gitlink SHA from the superproject index; check whether it exists or write a small helper using `git ls-tree HEAD <subPath>` to extract the SHA without ambiguity.

### Step 4: Unit tests

Add `internal/sync/validate_update_test.go` (host `t.TempDir()`, no git network calls needed since `validateUpdate` just parses text output):

```go
func TestValidateUpdateDriftIsError(t *testing.T) {
    // Simulate 'git submodule status' output with a '+' prefix line.
    cases := []struct{
        name   string
        output string
        wantErr bool
    }{
        {"clean", " abc1234 projects/foo (heads/main)", false},
        {"not-initialized", "-abc1234 projects/foo", true},
        {"drift", "+abc1234 projects/foo (heads/main)", true},
    }
    // ... inject the output via a testable variant of validateUpdate that
    // accepts a reader instead of running git.
}
```

Structure the test so that the validateUpdate logic can be called with injected output (extract a `parseSubmoduleStatus(r io.Reader) error` helper or pass the raw string to a private helper used by the test).

### Step 5: Container test

In `tests/integration/sync_drift_test.go`:

1. Create a bare origin with two commits: SHA-A (initial) and SHA-B (one more).
2. Clone as a superproject, add the submodule at SHA-A, commit the gitlink at SHA-A.
3. In the submodule, `git fetch origin` so SHA-B is known locally but the local `main` branch still points at SHA-A.
4. Run `camp sync` against the superproject.
5. Assert: result reports drift for the submodule (non-success or warning present).
6. Run `camp refs sync` (or equivalent refs-sync step).
7. Assert: the gitlink in the superproject index has NOT regressed to SHA-A (it should still be SHA-A since sync did not move it backwards, or it should be SHA-B if the fast-forward succeeded; it must not regress).

## Edge Cases

- **macOS `/private/var`**: `subDir` paths need `filepath.EvalSymlinks` before SHA comparison if the path appears in git output in resolved form. The `RevParse` output is a raw SHA so no path comparison is needed there; path comparisons in test assertions need the EvalSymlinks treatment.
- **Detached HEAD submodule**: `CheckedOutBranch` will be empty string or `"HEAD"` from `DetectDefaultBranch` returning an error; the drift check should skip when no branch name is available and log a warning only.
- **`git submodule status` line parsing**: The `+` line format is `+<sha> <path> (<describe>)`. The SHA after `+` is what was expected by the gitlink; the current worktree HEAD is the one that differs. Be precise about which SHA goes in the error message.

## Out of Scope

- Fixing the sync exit code table (sequence 10 covers the broader exit code standardization).
- Fixing `camp refs sync` unscoped commit (sequence 07 covers commit scoping).
- Changing the semantics of `--force` in sync (sequence 02 covers consent for destructive sync; this task only makes drift visible).

## Verified Negatives (C-3, do not re-fix)

The sync orphan-gitlink cleanup (`cleanOrphanedGitlinks` and `RemoveOrphanedGitlinks`) was verified clean in the second pass and must not be touched here.

## Done When

- [ ] `CheckoutDefaultBranch` call in `initSubmodules` captures both return values; checkout errors are recorded per-submodule
- [ ] `validateUpdate` no longer silently skips `+` drift; every `+` line returns an error or appended warning
- [ ] Unit test for `validateUpdate` (or its extracted parser) covers `-`, `+`, and ` ` cases
- [ ] Container test passes: stale-local-main scenario produces non-success; subsequent refs sync does not regress the pointer
- [ ] Both-profile gate passes after these changes