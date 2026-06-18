---
fest_type: task
fest_id: 01_flaky_test_fixes.md
fest_name: flaky_test_fixes
fest_parent: 12_test_hygiene_and_docs
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.804874-06:00
fest_updated: 2026-06-15T15:41:19.473673-06:00
fest_tracking: true
---


# Task: Flaky Test Fixes and Suite Hygiene

## Objective

Eliminate the named flaky-test patterns (sleep-race goroutines, mtime sleeps, double Reset, manual Setenv/Chdir boilerplate, brittle human-string Contains assertions) and start the host-git allowlist burn-down from `internal/git/commit/commit_test.go`.

Findings: R34 (TEST-6, TEST-8); TEST-5 burn-down start.

## Requirements

- [x] **Goroutine sleep-race lock release** (TEST-6): `internal/git/commit_test.go:940` and `internal/git/retry_test.go:30` release an index.lock from a goroutine after a fixed wall-clock sleep. Convert to channel-based release so the lock is dropped precisely when the test wants to signal readiness, not after an arbitrary delay.
- [x] **mtime sleeps** (TEST-6): five tests sleep 1-2ms to force distinct timestamps. Replace with `os.Chtimes` to set mtime explicitly without sleeping:
  - `internal/config/campaign_test.go:341` (context timeout test sleeps 1ms to ensure timeout; this is a context cancellation test not an mtime test, note below)
  - `internal/config/registry_test.go:400` (sleeps 1ms before `UpdateLastAccess` to get a distinct timestamp)
  - `internal/project/add_test.go:168` (context timeout test, same pattern as campaign_test)
  - `internal/workflow/schema_test.go:277` (context timeout test)
  - `internal/nav/index/cache_test.go:490` (sleeps 10ms before a forced rebuild to get a different BuildTime)
- [x] **Double Reset per checkout** (TEST-8): `tests/integration/main_test.go` calls `c.Reset()` on checkout (line 122) and again in the cleanup closure (line 131 area). The return-side Reset is redundant because the next checkout resets before use. Drop the return-side Reset from the `t.Cleanup` closure.
- [x] **Manual Setenv pairs to t.Setenv** (TEST-8): `internal/editor/editor_test.go` contains approximately 17 manual `os.Getenv`/`os.Setenv`/`os.Unsetenv` save-restore pairs across six test functions (lines 13-273 area). Convert all of them to `t.Setenv`, which auto-restores on test cleanup and is parallel-safe.
- [x] **t.Chdir modernization** (TEST-8): zero `t.Chdir` uses exist despite Go 1.25.6. `cmd/camp/id_test.go:86` defines `chdirForTest` (a hand-rolled helper that saves cwd, calls `os.Chdir`, and defers restore). Replace this helper with `t.Chdir`. Audit for any additional hand-rolled chdirForTest-style helpers and replace them too.
- [x] **Contains assertion conversion** (TEST-8): `tests/integration/` has 363 `Contains(t, output, "...")` assertions on human-readable strings. Converting all 363 is out of scope. Scope: convert the named starter set listed in the Implementation section to `--json` envelope assertions. These are the most brittle because they assert on multi-word prose that copy edits will break.
- [x] **Host-git burn-down start** (TEST-5): `internal/git/commit/commit_test.go` has 105 host-git call sites (the largest file in the 34-file ratchet allowlist). Migrate the most safety-critical flows from that file to `tests/integration/` (container-based, `//go:build integration`). The ratchet allowlist (`lint-no-host-fs-tests` in the justfile) must shrink by at least one entry by the end of this task. The full 34-file burn-down is out of scope; only this file's critical flows are in scope.

## Implementation

### Background

The ratchet gate `lint-no-host-fs-tests` (`.justfiles/lint.just`) blocks new host-git test files from growing beyond the 34-file allowlist. The allowlist has not shrunk since introduction (CW0003-tests-01). The campaign standard for unit tests is host `t.TempDir()` (documented exception for camp/fest unit suites per reference_fest_test_convention); the standard for destructive-path coverage is the container harness under `tests/integration/`.

Verify all file:line numbers against the worktree at `projects/worktrees/camp/camp-hardening` before editing; lines may have drifted from review commit `5c22d2b`.

### Step 1: Goroutine sleep-race fixes

**Background:** Both tests create an index.lock, start a goroutine that sleeps a fixed duration then closes and removes the lock, and then expect the retry loop to succeed. On a loaded machine the goroutine may not release within its window.

**File: `internal/git/commit_test.go` (around line 921, function `TestStage_WaitsForBriefActiveLock`)**

Locate the goroutine:
```go
go func() {
    time.Sleep(300 * time.Millisecond)
    _ = f.Close()
    _ = os.Remove(lockPath)
}()
```

Replace with a channel-signaled release:

```go
ready := make(chan struct{})
go func() {
    <-ready
    _ = f.Close()
    _ = os.Remove(lockPath)
}()

// Signal release before calling StageAll so the goroutine races
// the retry loop, but the outcome is deterministic: by the time
// StageAll polls, the lock is gone.
close(ready)
```

**File: `internal/git/retry_test.go` (around line 29, function `TestWithLockRetry_WaitsForActiveLockRelease`)**

Same pattern -- locate the goroutine sleeping 150ms and replace identically:

```go
ready := make(chan struct{})
go func() {
    <-ready
    _ = f.Close()
    _ = os.Remove(lockPath)
}()
close(ready)
```

### Step 2: mtime sleep replacements

**Background:** Five locations use `time.Sleep` in test setup. The campaign_test, add_test, and schema_test sleeps are for context-cancellation tests (sleep forces the context past its 1ns timeout before the function call). The registry_test sleep forces a distinct `time.Time` before updating LastAccess. The cache_test sleep forces a distinct BuildTime.

**`internal/config/campaign_test.go:341` (TestLoadCampaignConfig_ContextTimeout)**

The sleep here is: `time.Sleep(1 * time.Millisecond)` -- after setting a 1ns timeout -- to guarantee expiry. This is a context-cancellation test, not an mtime test. The right fix is to cancel the context explicitly rather than sleeping:

```go
func TestLoadCampaignConfig_ContextTimeout(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    cancel() // cancelled before any call

    _, err := LoadCampaignConfig(ctx, "/some/path")
    if !errors.Is(err, context.Canceled) {
        t.Errorf("LoadCampaignConfig() error = %v, want context.Canceled", err)
    }
}
```

Apply the same transformation to `internal/project/add_test.go:168` (`TestAdd_ContextTimeout`) and `internal/workflow/schema_test.go:277` (`TestFindSchema_ContextTimeout`): cancel explicitly instead of sleeping past a nano-timeout.

**`internal/config/registry_test.go:400` (TestRegistry_UpdateLastAccess)**

Locate:
```go
time.Sleep(1 * time.Millisecond)
reg.UpdateLastAccess("test-id")
```

Replace the sleep by manipulating the timestamp directly. The goal is just that `c2.LastAccess.After(initial)` holds. Instead of sleeping, record `initial` as a time one millisecond in the past:

```go
// Force initial to be in the past so any update is guaranteed After.
c1, _ := reg.GetByID("test-id")
initial := c1.LastAccess.Add(-2 * time.Millisecond)
reg.UpdateLastAccess("test-id")
```

If the registry sets LastAccess to `time.Now()` in UpdateLastAccess, this approach is correct without any sleep.

**`internal/nav/index/cache_test.go:490` (force rebuild BuildTime check)**

Locate the `time.Sleep(10 * time.Millisecond)` before the forced rebuild call. The test asserts that `idx2.BuildTime` differs from `idx1.BuildTime`. BuildTime is set from a real clock call inside `GetOrBuild`. Use `os.Chtimes` to advance the mtime of the root directory, which forces a stale detection triggering a new build with a new timestamp:

```go
// Advance the root mtime so the index sees it as stale.
now := time.Now().Add(time.Second)
if err := os.Chtimes(root, now, now); err != nil {
    t.Fatalf("Chtimes: %v", err)
}
```

Remove the `time.Sleep(10 * time.Millisecond)` line.

### Step 3: Drop return-side Reset in integration TestMain

**File: `tests/integration/main_test.go`** (verified: lines 114-140 area)

`GetSharedContainer` currently calls `c.Reset()` at checkout (line 122 area) and registers a cleanup that calls `c.Reset()` again before returning the container to the pool. The return-side Reset is redundant: the next caller's checkout Reset covers it.

Find the `t.Cleanup` closure that contains `_ = c.Reset()` followed by `containerPool <- c`:

```go
t.Cleanup(func() {
    _ = c.Reset()      // <-- REMOVE this line
    containerPool <- c
})
```

Remove only the `_ = c.Reset()` line. Leave `containerPool <- c` in place. Do not change the checkout Reset at line 122.

### Step 4: editor_test.go - convert manual Setenv to t.Setenv

**File: `internal/editor/editor_test.go`** (681 lines total)

There are approximately 6 test functions, each with the pattern:

```go
origEditor := os.Getenv("EDITOR")
origVisual := os.Getenv("VISUAL")
defer func() {
    if origEditor != "" {
        os.Setenv("EDITOR", origEditor)
    } else {
        os.Unsetenv("EDITOR")
    }
    if origVisual != "" {
        os.Setenv("VISUAL", origVisual)
    } else {
        os.Unsetenv("VISUAL")
    }
}()
```

And inside table-driven sub-tests:
```go
os.Setenv("EDITOR", tt.envEditor)
os.Setenv("VISUAL", tt.envVisual)
```

Convert every occurrence to `t.Setenv`:

At the top-level test function, replace the save/defer pattern with nothing (no save/restore needed; t.Setenv handles it).

Inside table-driven sub-tests, replace:
```go
os.Setenv("EDITOR", tt.envEditor)
os.Setenv("VISUAL", tt.envVisual)
```
with:
```go
t.Setenv("EDITOR", tt.envEditor)
t.Setenv("VISUAL", tt.envVisual)
```

Where `os.Unsetenv` is used to clear a variable, use `t.Setenv("EDITOR", "")`. Note: `t.Setenv` with an empty string sets the var to empty rather than unsetting it. If any test asserts on `os.Getenv("EDITOR") == ""` versus `!os.LookupEnv("EDITOR")` (var absent), adjust to clear correctly with a local `t.Cleanup(func() { os.Unsetenv("EDITOR") })` for that case. Inspect each usage.

After conversion, add `t.Parallel()` at the top of each top-level test function in this file (the convert-to-t.Setenv removes the global-state hazard that made parallel unsafe).

### Step 5: t.Chdir modernization

**File: `cmd/camp/id_test.go:86`** (function `chdirForTest`)

Locate the helper:
```go
func chdirForTest(t *testing.T, dir string) {
    t.Helper()
    origDir, err := os.Getwd()
    if err != nil {
        t.Fatalf("getwd: %v", err)
    }
    if err := os.Chdir(dir); err != nil {
        t.Fatalf("chdir %s: %v", dir, err)
    }
    campaign.ClearCache()
    t.Cleanup(func() {
        if err := os.Chdir(origDir); err != nil {
            t.Fatalf("restore cwd: %v", err)
```

Replace the body with:
```go
func chdirForTest(t *testing.T, dir string) {
    t.Helper()
    t.Chdir(dir)
    campaign.ClearCache()
}
```

`t.Chdir` (Go 1.24+) saves the original directory and restores it on cleanup automatically.

Audit the rest of the codebase for other hand-rolled chdirForTest-style helpers (`grep -rn "Getwd\|os.Chdir" --include="*_test.go" "$wt/"`) and replace each one found.

### Step 6: Contains assertion starter set conversion

**Scope**: convert the following named assertions in `tests/integration/` to `--json` envelope assertions. Do not convert all 363.

| File | Approximate line | Assertion string | Replacement approach |
|---|---|---|---|
| `project_linked_test.go` | ~28 | `"Committed changes to git"` | Run with `--json`, assert on envelope `success: true` field |
| `create_test.go` | ~37 | `"Campaign Initialized"` | `camp init --json` is not yet supported; skip this one and pick the next easiest |
| `workitem_commit_test.go` | any two | human commit confirmation strings | Use `camp commit --json` assertions if that flag exists; otherwise `--dry-run` output format |

For each assertion converted, the pattern is:

1. Change the command invocation to add `--json` (or `--format json`).
2. Unmarshal the output into the relevant JSON contract struct (e.g., `jsoncontract.Envelope`).
3. Assert on `envelope.Success` or a specific field instead of `strings.Contains(output, "human string")`.

Reference: `tests/integration/json_error_envelope_test.go` for existing examples of this pattern.

If a command does not yet support `--json`, skip it and pick the next most brittle assertion in the starter set. Record which assertions were converted and which were skipped with reason in the task notes.

### Step 7: Host-git burn-down - internal/git/commit/commit_test.go

**Background:** This file has 105 host-git call sites, the largest in the 34-file allowlist. The allowlist lives in the lint recipe (`lint-no-host-fs-tests` in `.justfiles/lint.just`). The goal is to migrate at least one critical flow to `tests/integration/` and remove the file from the allowlist (or shrink it to a smaller set if the file retains non-git tests).

**Critical flows to migrate** (the most safety-relevant):

1. `TestCommit_StagedFilesOnly` or equivalent test that verifies `--only` scoping does not sweep unstaged changes. This is the guard for N-4/N-5 scope correctness.
2. Any test verifying that committing with `Only: []string{path}` commits exactly `path` and leaves pre-staged unrelated files intact.

Migration steps:

1. Identify the test functions in `internal/git/commit/commit_test.go` that exercise the `Only`-scoped commit path (grep for `Only:` assignments in the test file).
2. Move those test functions to `tests/integration/` as a new file `git_commit_scope_test.go` with `//go:build integration`. Adapt them to use the container harness (`GetSharedContainer`, `RunCamp`, etc.) rather than direct `initTestRepo` helpers.
3. Delete the moved functions from `internal/git/commit/commit_test.go`.
4. Update the allowlist in `.justfiles/lint.just`. If `internal/git/commit/commit_test.go` still has remaining non-git tests, it may stay in the allowlist in reduced form; if all remaining tests also use git, remove the entry entirely.

**Out of scope**: migrating all 105 call sites. The full burn-down of the 34-file allowlist is a post-festival task. Only the `Only`-scoped flows and allowlist shrinkage are in scope here.

### Edge cases and notes

- The context-cancellation tests (campaign_test, add_test, schema_test) currently use a 1ns timeout + 1ms sleep. The explicit-cancel approach above is cleaner, but verify the function under test actually checks `ctx.Err()` before I/O, not after. If the function reads context only at entry, the 1ns timeout approach does test that; explicit cancel tests the same thing more reliably.
- For `cache_test.go:490`: `os.Chtimes` on the root directory manipulates mtime; verify that `GetOrBuild` uses `os.Stat(root).ModTime()` in its staleness check (it does, per `internal/nav/index/cache.go:96-168`).
- The `t.Chdir` change in `id_test.go` also affects `campaign.ClearCache()` ordering: ensure the cache clear still happens right after the directory change.
- For the double-Reset removal: confirm the pool contract still holds. The container returns to the pool once (from the cleanup), and is reset on the next checkout. This is the intended design per the container pool comment in `main_test.go:113`.

## Done When

- [x] Both-profile gate green twice in succession: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`
- [x] `TestStage_WaitsForBriefActiveLock` and `TestWithLockRetry_WaitsForActiveLockRelease` pass 10 consecutive runs without flaking (`go test -count=10 -run TestStage_WaitsForBriefActiveLock ./internal/git/... && go test -count=10 -run TestWithLockRetry_WaitsForActiveLockRelease ./internal/git/...`)
- [x] No `time.Sleep` in the five named test sites (grep confirms)
- [x] `internal/editor/editor_test.go` contains zero `os.Setenv`/`os.Unsetenv`/`os.Getenv` calls (all converted to `t.Setenv`)
- [x] `chdirForTest` helper in `cmd/camp/id_test.go` replaced with `t.Chdir`
- [x] At least three Contains assertions converted to JSON envelope assertions in `tests/integration/`
- [x] Host-git ratchet allowlist in `.justfiles/lint.just` has at least one fewer entry than before (or `internal/git/commit/commit_test.go` has fewer host-git call sites if retained in reduced form)
- [x] `just test integration` passes for any new container test added

## Completion Notes

- Converted the two lock-release tests to channel-driven release and hardened stale-lock removal against a disappearing lock between classification and remove.
- Replaced the named context-timeout sleeps with already-expired deadlines; replaced timestamp sleeps in registry/cache tests with direct time manipulation and `os.Chtimes`.
- Removed the return-side integration container `Reset`; checkout still resets before reuse.
- Converted editor env setup to `t.Setenv`. Did not add `t.Parallel` because Go disallows `t.Setenv` in parallel tests due process-global environment mutation.
- Replaced `cmd/camp/id_test.go` `chdirForTest` with `t.Chdir`; also converted isolated cwd helpers in `cmd/camp/intent/json_contract_test.go`, `internal/commands/flow/items_test.go`, and `internal/commands/workflow/create_test.go`. Left the package-scoped `internal/commands/workitem` helper on explicit restore after package tests showed the restore function is part of that package's test contract.
- Converted three brittle `workitem commits` human-output assertions to parsed `--json` assertions. `camp init --json` and the named project-link command remain unsupported for this starter set, so they were skipped with this substitution.
- Moved the pre-staged-unrelated-file selective commit coverage into the integration crawl test and removed the duplicate host-git unit test. The host-fs allowlist remains at 31 legacy files because `internal/git/commit/commit_test.go` still has legacy host-git coverage, but that retained file is reduced from 105 to 100 direct git command sites.
- Verification: both-profile build/unit/lint gates passed twice before the final helper audit edits; after those edits, affected package tests passed, `just test unit` passed 81 packages / 2522 tests, `just lint` passed, `git diff --check` passed, lock tests passed `-count=10`, targeted integration tests passed, and full `just test integration` passed 373/373 tests.