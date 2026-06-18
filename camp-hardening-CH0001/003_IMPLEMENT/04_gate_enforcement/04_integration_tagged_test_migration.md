---
fest_type: task
fest_id: 04_integration_tagged_test_migration.md
fest_name: integration_tagged_test_migration
fest_parent: 04_gate_enforcement
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.656802-06:00
fest_updated: 2026-06-14T21:17:30.618208-06:00
fest_tracking: true
---


# Task: Delete or Migrate the Five Dead //go:build integration Host Test Files

## Objective

Remove the five `//go:build integration` host test files that have never run under any gate, migrate worthwhile coverage into `tests/integration/`, and add `go vet -tags=integration ./...` to `just lint` so tagged code can never silently rot again.

## Requirements

- [x] `cmd/camp/commit_integration_test.go` is deleted (does not compile; `syncParentRef` undefined at :317; coverage duplicated by `tests/integration/workitem_commit_test.go`).
- [x] `cmd/camp/pull_integration_test.go` is deleted or its non-duplicated tests migrated to `tests/integration/`; review against existing `tests/integration/pull_test.go` and `staging_test.go`.
- [x] `cmd/camp/status_integration_test.go` is deleted (2 tests cover only raw `git status --short` git behavior, not camp commands; no migration value).
- [x] `cmd/camp/refs/commands_integration_test.go` is deleted or its non-duplicated tests migrated to `tests/integration/`; check against existing container commit and refs tests.
- [x] `internal/git/lock_integration_test.go` is deleted or migrated; 2 tests cover `CleanStaleLocks` and `FindIndexLocks` using host `t.TempDir()` with real git; evaluate whether the container harness already covers the same behavior via `tests/integration/pull_test.go`.
- [x] After deletions/migrations, `go vet -tags=integration ./...` exits 0.
- [x] `go vet -tags=integration ./...` is added to `just lint` (and therefore also to `just gate-fast` which calls `just lint`).
- [x] Files removed from the allowlist in `justfile`'s `lint-no-host-fs-tests` recipe for the deleted files.

## Implementation

### Background

TEST-2 (findings-testing.md) identifies five files carrying `//go:build integration` outside `tests/integration/`. No gate ever executes them: the unit runner passes no integration tag, the container integration runner targets only `tests/integration/` (`.justfiles/test.just` line 67 passes `-tags=integration ./tests/integration/...`), and there is no CI. The compile error in `commit_integration_test.go:317` (`undefined: syncParentRef`) has existed since approximately 2026-03-13 when `syncParentRef` moved to `cmd/camp/project`. These files create the illusion of coverage on the single most user-trusted flow.

The second-pass verification (section 1, row 2) confirmed the compile error live: `go vet -tags=integration ./cmd/camp/...` fails with `cmd/camp/commit_integration_test.go:317:12: undefined: syncParentRef`.

### Verified file:line targets in the worktree at bc2ad1f

**`cmd/camp/commit_integration_test.go`** (370 lines, 10 tests):

Tests: `TestIntegration_CommitBasic`, `TestIntegration_CommitNoChanges`, `TestIntegration_CommitWithStaleLock`, `TestIntegration_CommitAllFunction`, `TestIntegration_ExecutorCleanLocks`, `TestIntegration_ExecutorHasChanges`, `TestIntegration_FindProjectRoot`, `TestIntegration_ProjectCommit_DefaultNoSync`, `TestIntegration_ProjectCommit_ExplicitSync`, `TestIntegration_SubmoduleDetection`.

Line 317: `if err := syncParentRef(ctx, campRoot, "projects/test-project", nil); err != nil {` - `syncParentRef` was in `cmd/camp/commit.go` when the test was written; it moved to `cmd/camp/project/commit.go` as part of the submodule sync separation. The function is no longer accessible from `package main`. This file cannot compile and has not compiled since approximately commit 752f0b7.

Coverage overlap: `tests/integration/workitem_commit_test.go` (15 tests) covers commit path including scoped commits, linked projects, staged commits, JSON contract, and failure modes. The basic commit, stale-lock, and executor behaviors tested here are covered more thoroughly by the container harness. **Decision: delete.**

**`cmd/camp/pull_integration_test.go`** (292 lines, 11 tests):

Tests: `TestIntegration_PullAll_UpToDate`, `TestIntegration_PullAll_PullsNewCommits`, `TestIntegration_PullAll_SkipsDetachedHEAD`, `TestIntegration_PullAll_SkipsNoUpstream`, `TestIntegration_PullAll_ContextCancellation`, `TestIntegration_PullAll_DivergentBranchesRebase`, `TestIntegration_PullAll_DivergentBranchesDefaultFails`, `TestIntegration_PullAll_ExplicitFfOnlyOverride`, `TestIntegration_PullAll_PassesThroughGitFlags`, `TestIntegration_PullAll_RebaseConflictAutoAborts`, `TestIntegration_PullAll_ReportsChangedRefs`.

Existing container coverage: `tests/integration/pull_test.go` has `TestPull_RemovesStaleIndexLock` and `TestPullAll_RemovesSubmoduleStaleIndexLock`. The divergent-branches, rebase, and ff-only flag tests have no container equivalent. `TestIntegration_PullAll_RebaseConflictAutoAborts` is directly related to the R7 finding (sequence 03). However, this file is in `package main` and uses host `t.TempDir()` and host `exec.Command("git", ...)`, which would grow the host-git allowlist. The `lint-no-host-fs-tests` rule blocks this if migrated as-is.

Migration decision: the divergent-branch and rebase tests have genuine coverage value. Migrate `TestIntegration_PullAll_DivergentBranchesRebase`, `TestIntegration_PullAll_DivergentBranchesDefaultFails`, `TestIntegration_PullAll_RebaseConflictAutoAborts`, and `TestIntegration_PullAll_ExplicitFfOnlyOverride` into `tests/integration/pull_test.go` using the container harness. The remainder (`UpToDate`, `PullsNewCommits`, `SkipsDetachedHEAD`, `SkipsNoUpstream`, `ContextCancellation`, `PassesThroughGitFlags`, `ReportsChangedRefs`) are either already covered by the existing container tests or are low-value status checks. Delete those. Then delete `cmd/camp/pull_integration_test.go`.

**`cmd/camp/status_integration_test.go`** (47 lines, 2 tests):

Tests: `TestIntegration_Status_DefaultHidesRefs`, `TestIntegration_Status_ShowRefsReveals`.

These tests call raw `git status --short` directly on a host temp repo, not through any camp command. They test git's `--ignore-submodules=all` behavior, which is a git implementation detail, not camp behavior. There is no camp surface being exercised. **Decision: delete.**

**`cmd/camp/refs/commands_integration_test.go`** (217 lines, 5 tests from the refs package):

Tests: `TestIntegration_DetectRefChanges`, `TestIntegration_RefsSyncAtomic`, `TestIntegration_RefsSyncNoOp`, `TestIntegration_RefsSyncSafetyCheck`, `TestIntegration_FilterRefPaths`.

This is in `package refs`, testing `DetectRefChanges`, `SyncRefs`, and `FilterRefPaths` directly via host git. The `RefsSyncAtomic` test exercises scoped commit semantics, which is directly related to R6/N-4 (sequence 07). However, the R6 fix in sequence 07 will need its own container tests. Migrate `TestIntegration_RefsSyncAtomic` and `TestIntegration_RefsSyncSafetyCheck` into `tests/integration/` (new file, e.g. `refs_sync_test.go`) using the container harness. Delete `TestIntegration_DetectRefChanges`, `TestIntegration_RefsSyncNoOp`, and `TestIntegration_FilterRefPaths` (pure logic, better as unit tests). Then delete `cmd/camp/refs/commands_integration_test.go`.

**`internal/git/lock_integration_test.go`** (98 lines, 2 tests):

Tests: `TestLockHandling_RealGitRepo`, `TestLockHandling_RealSubmodule`.

Both use host `t.TempDir()` and real `exec.Command("git", ...)`. They test `CleanStaleLocks` and `FindIndexLocks` directly. The container harness indirectly exercises stale lock cleanup via `tests/integration/pull_test.go:TestPull_RemovesStaleIndexLock` and `TestPullAll_RemovesSubmoduleStaleIndexLock`. The unit-level lock logic (`CleanStaleLocks`, `FindIndexLocks`) is pure function testing and could be tested with host `t.TempDir()` without `exec.Command("git", ...)` - but the current tests do use `exec.Command("git", ...)`. To avoid growing the allowlist, delete the file. The container pull tests provide the behavioral coverage that matters. **Decision: delete.**

### Step-by-step approach

**Step 1. Confirm the compile error.**

```bash
cd projects/worktrees/camp/camp-hardening
go vet -tags=integration ./cmd/camp/...
```

Expected: fails with `cmd/camp/commit_integration_test.go:317:12: undefined: syncParentRef`. This confirms the file must be deleted (it cannot be fixed without importing from `cmd/camp/project`, which would require a package refactor out of scope here).

**Step 2. Delete the four files to delete immediately.**

```bash
rm cmd/camp/commit_integration_test.go
rm cmd/camp/status_integration_test.go
rm internal/git/lock_integration_test.go
```

(Do not delete `pull_integration_test.go` or `refs/commands_integration_test.go` yet; migrate first.)

**Step 3. Migrate pull tests to the container harness.**

In `tests/integration/pull_test.go`, add four new test functions after the existing tests. Follow the container harness conventions established by `GetSharedContainer(t)`, `tc.Shell(t, ...)`, `tc.RunCampInDir(...)`:

```go
func TestPullAll_DivergentBranchesRebase(t *testing.T) {
    tc := GetSharedContainer(t)

    const campaignDir = "/campaigns/pull-diverge-rebase"
    setupPullLockRootRepo(t, tc, campaignDir)

    tc.Shell(t, `
        git clone /test/root-remote.git /test/root-seed-diverge
        cd /test/root-seed-diverge
        printf 'remote line' > remote.txt
        git add remote.txt
        git commit -m 'remote diverge'
        git push origin main
    `)

    // Create a local diverging commit
    tc.Shell(t, fmt.Sprintf(`
        cd %s
        printf 'local line' > local.txt
        git add local.txt
        git commit -m 'local diverge'
    `, campaignDir))

    output, err := tc.RunCampInDir(campaignDir, "pull", "--rebase")
    require.NoError(t, err, "pull --rebase should succeed on divergent branches; output:\n%s", output)
    exists, err := tc.CheckFileExists(campaignDir + "/remote.txt")
    require.NoError(t, err)
    require.True(t, exists, "remote file should be present after rebase pull")
}

func TestPullAll_DivergentBranchesDefaultFails(t *testing.T) {
    tc := GetSharedContainer(t)

    const campaignDir = "/campaigns/pull-diverge-default"
    setupPullLockRootRepo(t, tc, campaignDir)

    tc.Shell(t, `
        git clone /test/root-remote.git /test/root-seed-divdef
        cd /test/root-seed-divdef
        printf 'remote line' > remote_def.txt
        git add remote_def.txt
        git commit -m 'remote diverge def'
        git push origin main
    `)

    tc.Shell(t, fmt.Sprintf(`
        cd %s
        printf 'local line' > local_def.txt
        git add local_def.txt
        git commit -m 'local diverge def'
    `, campaignDir))

    _, err := tc.RunCampInDir(campaignDir, "pull", "--ff-only")
    require.Error(t, err, "ff-only pull should fail on divergent branches")
}

func TestPullAll_FfOnlyOverride(t *testing.T) {
    tc := GetSharedContainer(t)

    const campaignDir = "/campaigns/pull-ffonly"
    setupPullLockRootRepo(t, tc, campaignDir)

    tc.Shell(t, `
        git clone /test/root-remote.git /test/root-seed-ff
        cd /test/root-seed-ff
        printf 'remote ff content' > ff.txt
        git add ff.txt
        git commit -m 'remote ff'
        git push origin main
    `)

    output, err := tc.RunCampInDir(campaignDir, "pull", "--ff-only")
    require.NoError(t, err, "ff-only pull should succeed when no divergence; output:\n%s", output)
    exists, err := tc.CheckFileExists(campaignDir + "/ff.txt")
    require.NoError(t, err)
    require.True(t, exists, "ff.txt should be present after ff-only pull")
}

func TestPullAll_RebaseConflictAutoAborts(t *testing.T) {
    tc := GetSharedContainer(t)

    const campaignDir = "/campaigns/pull-rebase-conflict"
    setupPullLockRootRepo(t, tc, campaignDir)

    tc.Shell(t, `
        git clone /test/root-remote.git /test/root-seed-conflict
        cd /test/root-seed-conflict
        printf 'remote version' > conflict.txt
        git add conflict.txt
        git commit -m 'remote adds conflict.txt'
        git push origin main
    `)

    // Local commit touching the same file
    tc.Shell(t, fmt.Sprintf(`
        cd %s
        printf 'local version' > conflict.txt
        git add conflict.txt
        git commit -m 'local adds conflict.txt'
    `, campaignDir))

    _, err := tc.RunCampInDir(campaignDir, "pull", "--rebase")
    require.Error(t, err, "rebase pull should fail on conflict")

    // After the failed pull, the repo must NOT be in a mid-rebase state
    output, code, execErr := tc.ExecCommand("git", "-C", campaignDir, "status")
    require.NoError(t, execErr)
    require.Equal(t, 0, code)
    require.NotContains(t, output, "rebase in progress", "repo should not be in mid-rebase state after auto-abort")
}
```

Note: these tests use `setupPullLockRootRepo` which is already defined in `tests/integration/pull_test.go`. If that helper does not create a writable remote, adjust the Shell scripts to push appropriately. Read `tests/integration/pull_test.go` in full before adding the new tests to confirm the helper creates the remote at `/test/root-remote.git`.

**Step 4. Delete cmd/camp/pull_integration_test.go.**

```bash
rm cmd/camp/pull_integration_test.go
```

**Step 5. Migrate refs tests to the container harness.**

Create `tests/integration/refs_sync_test.go` with the two worthwhile tests:

```go
//go:build integration
// +build integration

package integration

import (
    "fmt"
    "testing"

    "github.com/stretchr/testify/require"
)

func TestIntegration_RefsSyncAtomic(t *testing.T) {
    tc := GetSharedContainer(t)

    const campaignDir = "/campaigns/refs-sync-atomic"
    _, err := tc.InitCampaign(campaignDir, "RefsSync", "product")
    require.NoError(t, err)
    require.NoError(t, tc.CreateGitRepo(campaignDir))

    // Add a submodule project
    tc.Shell(t, fmt.Sprintf(`
        mkdir -p /test/sub-remote.git
        git init --bare /test/sub-remote.git
        git clone /test/sub-remote.git /test/sub-seed
        cd /test/sub-seed
        printf 'init' > README.md
        git add README.md
        git commit -m 'init sub'
        git push origin main
        cd %s
        GIT_ALLOW_PROTOCOL=file git submodule add /test/sub-remote.git projects/sub
        git commit -m 'add sub'
    `, campaignDir))

    // Advance the submodule
    tc.Shell(t, `
        cd /test/sub-seed
        printf 'advance' > advance.txt
        git add advance.txt
        git commit -m 'advance sub'
        git push origin main
        cd /campaigns/refs-sync-atomic/projects/sub
        git pull origin main
    `)

    // camp refs-sync should update the campaign root pointer atomically
    output, err := tc.RunCampInDir(campaignDir, "refs-sync")
    require.NoError(t, err, "refs-sync should succeed; output:\n%s", output)

    // The campaign root should now have a commit recording the updated pointer
    logOut, code, execErr := tc.ExecCommand("git", "-C", campaignDir, "log", "--oneline", "-1")
    require.NoError(t, execErr)
    require.Equal(t, 0, code)
    require.NotEmpty(t, logOut, "refs-sync should have created a commit")
}

func TestIntegration_RefsSyncSafetyCheck(t *testing.T) {
    tc := GetSharedContainer(t)

    const campaignDir = "/campaigns/refs-sync-safety"
    _, err := tc.InitCampaign(campaignDir, "RefsSyncSafety", "product")
    require.NoError(t, err)
    require.NoError(t, tc.CreateGitRepo(campaignDir))

    // With no submodule drift, refs-sync should be a no-op (exit 0, no new commit)
    logBefore, _, _ := tc.ExecCommand("git", "-C", campaignDir, "log", "--oneline", "-1")
    output, err := tc.RunCampInDir(campaignDir, "refs-sync")
    require.NoError(t, err, "refs-sync no-op should succeed; output:\n%s", output)
    logAfter, _, _ := tc.ExecCommand("git", "-C", campaignDir, "log", "--oneline", "-1")
    require.Equal(t, logBefore, logAfter, "refs-sync with no drift should not create a commit")
}
```

**Step 6. Delete cmd/camp/refs/commands_integration_test.go.**

```bash
rm cmd/camp/refs/commands_integration_test.go
```

**Step 7. Update the lint-no-host-fs-tests allowlist.**

The allowlist in the root `justfile` (line 78) includes references to the deleted files. Remove the following from the allowlist string:

- `./cmd/camp/commit_integration_test.go`
- `./cmd/camp/refs/commands_integration_test.go`
- `./internal/git/lock_integration_test.go`

The `./cmd/camp/pull_integration_test.go` entry is also in the allowlist but was named under `cmd/camp/pull_integration_test.go` - check whether it appears and remove it too.

Read the justfile allowlist line before editing to get the exact current string.

**Step 8. Add go vet -tags=integration to just lint.**

The `lint` recipe in the root justfile runs golangci-lint then calls `just lint-no-host-fs-tests`. Add one more line at the end:

```just
lint:
    #!/usr/bin/env sh
    set -eu
    base="${GOLANGCI_LINT_BASE:-origin/main}"
    if git rev-parse --verify "$base" >/dev/null 2>&1; then
        golangci-lint run --new-from-merge-base "$base" ./...
    else
        echo "lint: $base not found; falling back to golangci-lint --new"
        golangci-lint run --new ./...
    fi
    just lint-no-host-fs-tests
    go vet -tags=integration ./...
```

The `go vet -tags=integration ./...` line is the ratchet: it compiles all integration-tagged code, so a future dead-reference like the `syncParentRef` failure will immediately surface.

**Step 9. Verify.**

```bash
go vet -tags=integration ./...
```

Must exit 0. Then:

```bash
just lint
```

Must exit 0 (includes the new vet line).

Run `just gate-fast` to confirm the full pipeline is clean after all deletions and migrations.

### Acceptance criteria detail

The `no //go:build integration files outside tests/integration/` requirement has one intentional exception: `internal/git/lock_integration_test.go` is deleted, but the allowlist entry `./internal/git/lock_integration_test.go` was in `lint-no-host-fs-tests`. After deletion, the allowlist no longer references it (remove the entry in step 7). Going forward, any `//go:build integration` file outside `tests/integration/` will compile under `go vet -tags=integration ./...` (now wired in lint); if it does not compile, lint fails.

### Out of scope

- Migrating every host-git test file to the container harness (that is the sequence 12 host-git allowlist burn-down per C-4).
- Fixing the underlying `syncParentRef` scoping issue (R6, sequence 07).
- Any GitHub Actions files (C-1).
- The double-Reset in container tests (sequence 12 per C-4).
- scc pinning in the container harness (deliberate per C-4).

## Done When

- [x] All requirements met
- [x] All five files are deleted or migrated; none remain in the worktree
- [x] `go vet -tags=integration ./...` exits 0
- [x] `go vet -tags=integration ./...` is wired into `just lint` (and therefore `just gate-fast`)
- [x] The three deleted filenames are removed from the `lint-no-host-fs-tests` allowlist in the root justfile
- [x] `just gate-fast` exits 0 after this task and task 01 of this sequence have both landed

## Verification

- Confirmed the stale tagged host tests were dead: `go vet -tags=integration ./cmd/camp/...` failed at `cmd/camp/commit_integration_test.go:317:12` with `undefined: syncParentRef`.
- Deleted `cmd/camp/commit_integration_test.go`, `cmd/camp/status_integration_test.go`, and `internal/git/lock_integration_test.go`.
- Migrated the valuable pull cases into `tests/integration/pull_test.go`: divergent rebase, divergent ff-only failure, explicit ff-only success, and rebase conflict auto-abort.
- Migrated the valuable refs-sync cases into `tests/integration/refs_sync_test.go`: atomic root pointer commit and no-drift safety/no-op behavior.
- Deleted `cmd/camp/pull_integration_test.go` and `cmd/camp/refs/commands_integration_test.go` after migration.
- Passed `go vet -tags=integration ./...`.
- Passed targeted migrated integration tests:

```bash
go test -count=1 -v -tags integration -timeout 10m ./tests/integration -run 'TestPullAll_(DivergentBranchesRebase|DivergentBranchesDefaultFails|FfOnlyOverride|RebaseConflictAutoAborts)|TestIntegration_RefsSync(Atomic|SafetyCheck)'
```

- Passed `just lint`; `lint-no-host-fs-tests` reported clean with 31 legacy files still on the allowlist.
- Passed `just gate-fast`; dev unit tests reported 81 packages and 2557/2557 tests passed.
- Passed `git diff --check`.