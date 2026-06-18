---
fest_type: task
fest_id: 01_test_runner_build_tags.md
fest_name: test_runner_build_tags
fest_parent: 04_gate_enforcement
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.65568-06:00
fest_updated: 2026-06-14T21:05:16.390708-06:00
fest_tracking: true
---


# Task: Make tasks.Test Honor BUILD_TAGS and Add just gate / just gate-fast Recipes

## Objective

Fix `internal/buildutil/tasks/test.go` so the test runner passes build tags to `go test` exactly as `tasks.Build` already does, add dev-profile test invocation to the justfile, and introduce the `just gate-fast` and `just gate` recipes that D005 specifies as the authoritative local gate.

## Requirements

- [x] `tasks.Test` in `internal/buildutil/tasks/test.go` passes `-tags <BUILD_TAGS>` to `go test` when `BUILD_TAGS` is set, using the same `appendBuildTags` helper that `tasks.Build` already uses.
- [x] `BUILD_TAGS=dev just test unit` causes dev-tagged test files to execute: `cmd/camp/release_profile_dev_test.go`, `cmd/camp/quest/helpers_test.go`, and `internal/quest/tui/create_model_test.go` must appear in the test run output.
- [x] `just gate-fast` recipe exists in the root justfile implementing D005 exactly: both-profile builds, `go vet ./...`, `go vet -tags dev ./...`, `go vet -tags=integration ./...`, lint in both profiles (`just lint` and `BUILD_TAGS=dev just lint`), and dev-profile unit tests.
- [x] `just gate` recipe exists running gate-fast plus stable-profile unit tests (`just test unit`), implementing the full C-2 both-profile matrix in one command.
- [x] No `.github/workflows/` files are added or modified (C-1).

## Implementation

### Background

BR-1 (findings-build-release.md) documents that `tasks.Test` runs `go test -count=1 -json -short ...` with no build tags while `tasks.Build` consults `BUILD_TAGS` via `appendBuildTags`. As a result, `release_profile_dev_test.go`, all `cmd/camp/quest/` tests, and `internal/quest/tui/create_model_test.go` run only when someone manually types `go test -tags dev`. This is exactly the surface that festival-app bundles and ships.

D005 specifies two recipes: `gate-fast` (the hook's default payload) and `gate` (the pre-release full matrix). Both are local only; no GitHub Actions.

### Verified file:line targets in the worktree at bc2ad1f

**`internal/buildutil/tasks/test.go`:**

- Line 84: `cmd := exec.Command("go", "test", "-count=1", "-json", "-short", "-timeout", "120s", pkg)` - this is the call that lacks build tags. The `Build` function at `build.go:98` uses `exec.Command("go", goVetArgs(pkg)...)` and `build.go:113` uses `exec.Command("go", goBuildArgs("-o", "/dev/null", pkg)...)`, both of which call `appendBuildTags` (defined at `build.go:47-56`).

- `appendBuildTags` reads `os.Getenv("BUILD_TAGS")` via `buildTags()` at `build.go:43-45`. It is in the same package (`tasks`), so `test.go` can call it directly after this change.

**Root `justfile`:**

- Lines 43-45: the existing `vet` recipe runs `go vet ./...` untagged.
- Lines 50-60: the existing `lint` recipe runs golangci-lint and then `just lint-no-host-fs-tests`.
- No `gate`, `gate-fast`, or `hooks install` recipes exist yet.

### Step-by-step approach

**Step 1. Read the current test.go command line.**

Open `internal/buildutil/tasks/test.go` in the worktree. Locate line 84:

```go
cmd := exec.Command("go", "test", "-count=1", "-json", "-short", "-timeout", "120s", pkg)
```

The fix is to prepend `-tags <BUILD_TAGS>` when the env var is set. The `appendBuildTags` helper already does this. Construct the args list the same way `goBuildArgs` does it for build:

Replace line 84 with:

```go
testArgs := append([]string{"test", "-count=1", "-json", "-short", "-timeout", "120s"}, appendBuildTags(pkg)...)
cmd := exec.Command("go", testArgs...)
```

`appendBuildTags(pkg)` returns `["-tags", "dev", pkg]` when `BUILD_TAGS=dev` and `[pkg]` when unset, so the resulting argv is either `go test -count=1 -json -short -timeout 120s -tags dev ./some/pkg` or the current untagged form.

Note: `discoverTestPackages` at line 290 calls `exec.Command("go", "list", "-f", "{{.TestGoFiles}}", pkg)` to check for test files. This list call runs without tags, which means a package that has ONLY dev-tagged test files (no untagged `_test.go`) will not appear in `testPackages`. For this festival's scope that is acceptable: the known dev-tagged test files (`release_profile_dev_test.go`, `quest/helpers_test.go`, `internal/quest/tui/create_model_test.go`) all coexist in packages that also have untagged test files, so they will be discovered. If a future package has only dev-tagged tests, `discoverTestPackages` would need to be updated. Document this as an out-of-scope edge case.

**Step 2. Verify the fix discovers dev test packages.**

After editing, from the worktree root run:

```bash
BUILD_TAGS=dev go run ./internal/buildutil test
```

Confirm in the output that `cmd/camp` appears (it contains `release_profile_dev_test.go`) and that the run count includes the dev-profile tests. A quick targeted check:

```bash
BUILD_TAGS=dev go test -tags dev -count=1 -short -run TestReleaseProfileDev ./cmd/camp
```

This should show `TestReleaseProfileDev_GendocsCommandHiddenButRegistered`, `TestReleaseProfileDev_DevCommandsRegistered`, `TestReleaseProfileDev_StableCommandsRegistered`, `TestReleaseProfileDev_BuildProfileRegistered` all passing.

Also confirm quest tests run:

```bash
BUILD_TAGS=dev go test -tags dev -count=1 -short ./cmd/camp/quest/...
```

**Step 3. Add gate-fast and gate recipes to the root justfile.**

The root justfile uses modular `mod` declarations plus top-level recipes. The existing style for top-level recipes (see `vet`, `lint`, `docs`, `build-camp`) uses simple recipe bodies. Add the two gate recipes after the existing `lint-no-host-fs-tests` recipe.

Append to the root `justfile` (after line 100 `install-tools`):

```just
# Run both-profile builds, vet (stable/dev/integration), lint both profiles, and dev unit tests.
# This is the pre-push hook's default payload (D005 gate-fast). See also: just gate.
gate-fast:
    #!/usr/bin/env sh
    set -eu
    echo "=== gate-fast: stable build ==="
    just build-camp
    echo "=== gate-fast: dev build ==="
    BUILD_TAGS=dev just build-camp
    echo "=== gate-fast: vet stable ==="
    go vet ./...
    echo "=== gate-fast: vet dev ==="
    go vet -tags dev ./...
    echo "=== gate-fast: vet integration ==="
    go vet -tags=integration ./...
    echo "=== gate-fast: lint stable ==="
    just lint
    echo "=== gate-fast: lint dev ==="
    BUILD_TAGS=dev just lint
    echo "=== gate-fast: unit tests dev (the profile festival-app ships) ==="
    BUILD_TAGS=dev just test unit
    echo "=== gate-fast: PASSED ==="

# Run the full both-profile gate: gate-fast plus stable unit tests (C-2 matrix).
# Required by release recipes before tagging; the per-sequence closing command for sequences 05-12.
# No GitHub Actions: all enforcement is local (C-1).
gate:
    #!/usr/bin/env sh
    set -eu
    just gate-fast
    echo "=== gate: unit tests stable ==="
    just test unit
    echo "=== gate: PASSED ==="

# Set git core.hooksPath to .githooks so the pre-push gate fires automatically.
# Idempotent: safe to run multiple times.
hooks-install:
    git config core.hooksPath .githooks
    @echo "Hooks installed: .githooks is now the active hooks directory."
    @echo "To verify: git config --get core.hooksPath"
```

Match the existing justfile style: `#!/usr/bin/env sh` shebang scripts for multi-line recipes, `@echo` for informational output on one-liners, no trailing whitespace. The recipe name `hooks-install` uses a hyphen to match the existing `install-tools`, `build-camp`, `lint-no-host-fs-tests` pattern.

**Step 4. Ordering note on go vet -tags=integration.**

When the implementer runs this task, `go vet -tags=integration ./...` will FAIL because `cmd/camp/commit_integration_test.go:317` references undefined `syncParentRef`. This vet failure is what task 04 of this sequence fixes by deleting or migrating that file. The recipe landing in task 01 is correct; the vet line inside it will pass only after task 04 cleans up the dead files. Run tasks in order: 01, then 04, then verify gate-fast exits 0 end to end. The hook (task 03) should be installed only after task 04.

**Step 5. Verify both gate recipes.**

After task 04 of this sequence has removed the dead integration-tagged files:

```bash
just gate-fast
just gate
```

Both must exit 0. Confirm that the `gate` output includes both dev and stable test runs.

### Edge cases

The `appendBuildTags` helper in `build.go` returns the extra args as a prefix before the package argument. The `testArgs` construction in step 1 appends `appendBuildTags(pkg)` last, which puts `-tags dev pkg` after the other flags. `go test` accepts `-tags` anywhere in the flag list, so order does not matter.

The `discoverTestPackages` function uses `go list -f {{.TestGoFiles}}` without tags. If `BUILD_TAGS=dev`, `go list` without tags may still show the untagged test files, causing the package to be included. The dev-tagged files will then compile into the test binary because the build command itself carries `-tags dev`. This is the correct behavior.

Implementation note: verification showed `cmd/camp/quest` and `internal/quest/tui` are dev-only packages, so untagged `go list ./...` excluded them entirely. The implementation therefore also routes package discovery and test-file discovery through `appendBuildTags` via `goListArgs`, so `BUILD_TAGS=dev just test unit` discovers 81 packages and includes the dev-only packages.

### Out of scope

- Updating `discoverTestPackages` to pass build tags to the `go list` call (only needed if a package has exclusively dev-tagged test files; none exist today).
- Any GitHub Actions files (C-1 hard constraint; mention of hosted CI is only as a prohibition).
- The double-Reset in container tests (that is sequence 12 flaky-test hygiene per C-4).
- scc pinning (deliberate documented choice per C-4).

## Done When

- [x] All requirements met
- [x] `BUILD_TAGS=dev just test unit` output includes tests from `cmd/camp` (quest and dev profile tests) with no "no test files" skip for dev-tagged packages that coexist with untagged ones
- [x] `just gate-fast` and `just gate` recipes exist in the root justfile and match D005 spec
- [x] Task-04 dependency documented: `go vet -tags=integration ./...` currently fails at `cmd/camp/commit_integration_test.go:317` (`undefined: syncParentRef`), and `just gate` exits 0 end to end after task 04 removes that known blocker.
- [x] Zero `.github/workflows/` files in the diff

## Verification

- `go test ./internal/buildutil/tasks -count=1`
- `BUILD_TAGS=dev just test unit` -> 81 packages, 2557/2557 tests passed; output includes `cmd/camp/quest` and `internal/quest/tui`.
- `go test -tags dev -count=1 -short -v -run 'TestReleaseProfileDev|TestAutoCommitQuest_RemoveCachedFailureReturnsError|TestQuestCreateModel_InitialState' ./cmd/camp ./cmd/camp/quest ./internal/quest/tui`
- `just build-camp`
- `BUILD_TAGS=dev just build-camp`
- `go vet ./...`
- `go vet -tags dev ./...`
- `go vet -tags=integration ./...` -> expected task-04 failure at `cmd/camp/commit_integration_test.go:317`.
- `just --dry-run gate-fast`
- `just --dry-run gate`
- `just lint`
- `BUILD_TAGS=dev just lint`
- `just test unit` -> 79 packages, 2537/2537 tests passed.