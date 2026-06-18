---
fest_type: task
fest_id: 05_container_reset_paths.md
fest_name: container_reset_paths
fest_parent: 04_gate_enforcement
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.657038-06:00
fest_updated: 2026-06-14T21:30:35.231705-06:00
fest_tracking: true
---


# Task: Fix Container Pool Reset for the .obey Path Move

## Objective

Update `TestContainer.Reset()` in `tests/integration/helpers_container.go` to remove the paths tests actually use between runs, eliminating inherited-state flakes from the registry path migration and per-test fixed work roots, and delete the manual workaround in `settings_test.go:16`.

## Requirements

- [x] `Reset()` removes `/root/.obey` (the current registry and config home per `create_test.go:15` constant `globalRegistryPath = "/root/.obey/campaign/registry.json"`).
- [x] `Reset()` removes the fixed `/tmp/create-*` work roots that `create_test.go` uses, preventing inherited state when a container is reused after those tests.
- [x] `Reset()` drops the legacy `/root/.config/camp` entry (the pre-migration config path; no longer relevant).
- [x] The manual `tc.Shell(t, "rm -rf /root/.obey/campaign")` at `tests/integration/settings_test.go:16` is deleted.
- [x] `just test integration` runs green at least twice back-to-back without the settings_test manual rm.

## Implementation

### Background

TEST-3 (findings-testing.md) documents the mismatch: `Reset()` at `tests/integration/helpers_container.go:21-35` removes `/test`, `/campaigns`, `/root/.config/camp`, and `/root/.camp`, but the registry now lives at `/root/.obey/campaign/registry.json` (confirmed in `create_test.go:15`, `registry_test.go:45`, `helpers_container.go:295`). This is the pre-migration config path. The migration to `.obey` was complete and no compat shims are needed (C-4). Reset was never updated.

Direct evidence the leak is a known problem: `tests/integration/settings_test.go:16` manually runs `rm -rf /root/.obey/campaign` before its assertion. This is an acknowledged workaround, not the intended design.

Fixed `/tmp/create-*` paths: `tests/integration/create_test.go` uses hard-coded `/tmp/create-happy` (line 27), `/tmp/create-nottymissing` (line 77), `/tmp/create-dryrun-base-nonexistent` (line 102), `/tmp/create-collision-empty` (line 137), `/tmp/create-collision-nonempty` (line 156), `/tmp/create-collision-existing` (line 181), `/tmp/create-creates-parent-auto` (line 201), and others. When a pooled container runs a `create_test` and then the container is returned and checked out by a different test, these `/tmp/create-*` directories persist. If the same container is reused for another create test, the `mkdir -p $base` step succeeds but the directory already contains state from the previous run, causing assertions about "directory does not exist yet" to fail non-deterministically.

The `/tmp/_camp_stdout`, `/tmp/_camp_stderr`, `/tmp/_camp_exitcode` files from `helpers_container.go:273-287` are already cleaned up per-invocation inside `RunCampInteractiveStepsInDir`, so they are not a Reset concern.

### Verified file:line targets in the worktree at bc2ad1f

**`tests/integration/helpers_container.go`:**

- Lines 21-35: the full `Reset()` function body. The `rm -rf` command currently is:
  `"rm -rf /test /campaigns /root/.config/camp /root/.camp 2>/dev/null; " + "mkdir -p /test /campaigns /root/.config/camp; sync"`

- Line 295: `WriteGlobalConfig` writes to `/root/.obey/campaign/config.json`, confirming the active config path.

**`tests/integration/settings_test.go`:**

- Line 16: `tc.Shell(t, "rm -rf /root/.obey/campaign")` - the manual workaround to remove.

**`tests/integration/create_test.go`:**

- Line 15: `const globalRegistryPath = "/root/.obey/campaign/registry.json"` - confirms the registry location.
- Lines 27, 77, 102, 137, 156, 181, 201, 231, 232, 266, 299: the fixed `/tmp/create-*` base paths used by create tests.

### Step-by-step approach

**Step 1. Read helpers_container.go:21-35 to confirm current state.**

Verify the exact current `rm -rf` string before editing. The source-of-truth read is required because line numbers may have drifted since bc2ad1f.

**Step 2. Update Reset() to clean the correct paths.**

Replace the `rm -rf` command string inside `Reset()`. The new command must:

- Remove `/root/.obey` (covers registry at `/root/.obey/campaign/registry.json` and config at `/root/.obey/campaign/config.json`).
- Remove `/test` and `/campaigns` (existing, keep these).
- Remove per-test fixed `/tmp/create-*` work roots.
- Drop `/root/.config/camp` (legacy pre-migration path, no tests use it).
- Drop `/root/.camp` (legacy, no tests use it).
- Re-create only the directories that tests expect to pre-exist: `/test` and `/campaigns`.

The new Reset body:

```go
func (tc *TestContainer) Reset() error {
    exitCode, _, err := tc.container.Exec(tc.ctx, []string{
        "sh", "-c",
        "rm -rf /test /campaigns /root/.obey /tmp/create-* 2>/dev/null; " +
            "mkdir -p /test /campaigns; sync",
    })
    if err != nil {
        return fmt.Errorf("failed to reset container: %w", err)
    }
    if exitCode != 0 {
        return fmt.Errorf("reset command failed with exit code %d", exitCode)
    }
    return nil
}
```

Key changes from the current version:

- Added `/root/.obey` to the rm list.
- Added `/tmp/create-*` glob to clear create_test fixed work roots.
- Removed `/root/.config/camp` from both the rm list and the mkdir re-creation (it was being recreated but is now abandoned).
- Removed `/root/.camp` from the rm list.
- `mkdir -p` now only creates `/test` and `/campaigns`.

Note: `/tmp/create-*` uses shell glob expansion inside `sh -c`, which works correctly with `rm -rf`. The 2>/dev/null suppresses "no such file" errors for any `/tmp/create-*` paths that do not exist on a given Reset.

**Step 3. Identify any other fixed /tmp paths that need Reset coverage.**

From the grep of integration tests above, the only other fixed `/tmp` paths are:

- `/tmp/commit_hook.sh`, `/tmp/env_dump` in `commit_tags_test.go:208-226` - these are test-specific temp files created and read within a single test function. The test does not assert absence at the start; no Reset coverage needed.
- `/tmp/workitem-commit-json-error-stdout`, `/tmp/workitem-commit-json-error-stderr` in `workitem_commit_test.go:526-527` - same pattern; single-test lifecycle.
- `/tmp/cancel.out`, `/tmp/link-*.out` in `workitem_commits_query_test.go:171` and `workitem_link_concurrent_test.go:42-49` - same pattern.
- `/tmp/workitem-link-json-error-stdout`, `/tmp/workitem-link-json-error-stderr` in `workitem_links_test.go:138-139` - same pattern.
- `/tmp/settings-campaigns` from `settings_test.go:27` - this is created by the test via `camp settings` and read back. The test asserts a specific path exists after the settings command runs. Reset does not need to clean it because the test uses a unique path per test, not a shared pre-existing path. But since `settings_test` manually cleans `/root/.obey/campaign` before running (the workaround this task removes), the fix in step 4 must ensure Reset covers it instead.

For `/tmp/settings-campaigns`: the test creates it after the `rm -rf /root/.obey/campaign` workaround in settings_test.go. After this task removes that workaround, Reset must leave the container in a state where `camp settings` can write to `/root/.obey/campaign/` again. Adding `/root/.obey` to Reset (step 2) ensures this directory is removed, so `camp settings` will recreate it fresh.

The `/tmp/settings-campaigns` path itself does not conflict across tests because it has a unique name and `camp settings` creates it during the test, not before. No additional Reset entry needed.

**Step 4. Delete the manual workaround in settings_test.go.**

In `tests/integration/settings_test.go`, remove line 16:

```go
tc.Shell(t, "rm -rf /root/.obey/campaign")
```

After this change, the test function `TestCampSettings_CampaignsDirEditAndClear` starts with just:

```go
func TestCampSettings_CampaignsDirEditAndClear(t *testing.T) {
    tc := GetSharedContainer(t)

    backFromGlobalSettings := ...
```

The Reset called by `GetSharedContainer` (via `main_test.go:122`) now removes `/root/.obey`, so the state the settings_test was manually cleaning is already gone when the test starts.

**Step 5. Verify with back-to-back integration runs.**

Run the integration suite twice to confirm no inherited-state flakes:

```bash
just test integration
just test integration
```

Both runs must be green. Specifically watch `TestCampSettings_CampaignsDirEditAndClear` and any `create_test` functions. If the suite is too slow to run twice on demand, at minimum run the affected test files directly:

```bash
# Verbose run of settings and create tests, twice
DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock" \
TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock \
go test -v -tags=integration ./tests/integration/... -run "TestCampSettings|TestCampCreate" -timeout 5m

DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock" \
TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock \
go test -v -tags=integration ./tests/integration/... -run "TestCampSettings|TestCampCreate" -timeout 5m
```

Both runs must pass. The second run proves the container was cleaned correctly between invocations.

### Edge cases

The `mkdir -p /root/.config/camp` in the current Reset is being dropped. No test currently asserts that `/root/.config/camp` pre-exists (it was the legacy config path). Dropping it from both rm and mkdir is safe.

The glob `/tmp/create-*` in the `sh -c` rm command will be expanded by `sh` inside the container. This is intentional shell-glob expansion, not Go string templating. The `2>/dev/null` suppresses the "no such file" error when no `/tmp/create-*` paths exist.

The double-Reset pattern noted in TEST-8 (`GetSharedContainer` resets on checkout at `main_test.go:122` and again on return at `main_test.go:131`) is NOT changed here. That is sequence 12 flaky-test hygiene per C-4. This task only fixes the rm/mkdir content, not the call count.

### Out of scope

- The double-Reset pattern (sequence 12 per C-4).
- scc pinning in the harness (deliberate per C-4).
- Containerizing host-fs tests from the 34-file allowlist (sequence 12 per C-4).
- Any GitHub Actions files (C-1).
- Adding per-test work-root generation (a future hygiene improvement; the fixed `/tmp/create-*` glob is the minimal fix for now).

## Done When

- [x] All requirements met
- [x] `tests/integration/helpers_container.go` Reset rm list includes `/root/.obey` and `/tmp/create-*`; does NOT include `/root/.config/camp` or `/root/.camp`; `mkdir -p` only creates `/test` and `/campaigns`
- [x] `tests/integration/settings_test.go` no longer contains the manual `rm -rf /root/.obey/campaign` line
- [x] `just test integration` runs green
- [x] `just test integration` run a second time immediately after still runs green (no inherited-state flake)

## Verification

- Updated `tests/integration/helpers_container.go` so `Reset()` now runs:

```sh
rm -rf /test /campaigns /root/.obey /tmp/create-* 2>/dev/null
mkdir -p /test /campaigns
```

- Removed the manual `/root/.obey/campaign` cleanup from `TestCampSettings_CampaignsDirEditAndClear`.
- The first full run after the reset change exposed a hidden stale-registry dependency in `TestProject_NotInCampaign`: it expected `campaign name required in non-interactive mode`, but a clean reset correctly produced `no campaigns registered`. Fixed the test by explicitly creating a registered campaign inside that test before asserting the non-interactive target-campaign error.
- Passed focused reproduction:

```bash
go test -v -tags=integration ./tests/integration -run '^TestProject_NotInCampaign$' -count=1 -timeout 5m
```

- Passed the required back-to-back full integration runs:

```bash
just test integration
# 1 suites  367/367 tests passed  179.01s

just test integration
# 1 suites  367/367 tests passed  174.43s
```