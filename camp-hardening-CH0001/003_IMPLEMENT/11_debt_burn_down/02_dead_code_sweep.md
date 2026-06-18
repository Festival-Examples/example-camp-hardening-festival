---
fest_type: task
fest_id: 02_dead_code_sweep.md
fest_name: dead_code_sweep
fest_parent: 11_debt_burn_down
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.787334-06:00
fest_updated: 2026-06-15T03:55:45.323544-06:00
fest_tracking: true
---


# Task: Dead Code Sweep (ARCH-5, ERR-6)

## Objective

Delete all symbols dead in both the stable and dev build profiles in one verified sweep, removing approximately 216 symbols (~2,000 lines), so task 03's extraction does not migrate dead code into internal packages.

## Requirements

- [x] This task executes STRICTLY BEFORE task 03 (cmd logic extraction); the dependency is load-bearing
- [x] `golang.org/x/tools/cmd/deadcode` run against BOTH tag sets before deletion confirms the inventory; re-run after deletion confirms none remain
- [x] Exclusions: `pkg/commitkit`'s 13 fest-facing exports and test fakes are NOT deleted
- [x] ERR-6 Get-prefix deletions (`GetUserEmail`, `GetUserIdentity`) ride in the same sweep
- [x] `internal/crawl/fake_prompt.go` methods moved to a `_test.go` file (not deleted)
- [x] Both-profile gate green after the single sweep commit
- [x] Note the `internal/config/allowlist.go:39-108` reconciliation with sequence 06 task 02 (see Implementation)

## Implementation

### Background

ARCH-5 identifies approximately 216 dead symbols across several clusters, all verified dead in BOTH stable and dev builds by the first-pass review. The worktree is at `bc2ad1f`. Before executing any deletion, re-verify each cluster still exists and is still dead by grep for references (do NOT run the deadcode tool speculatively during planning; run it during execution). The key verification rule: if any caller references a symbol, it is not dead and must not be deleted.

R29 mandates this task runs BEFORE cmd logic extraction (task 03) to avoid migrating dead code into new internal packages.

### Step 1: Run deadcode to confirm inventory (execution step)

At execution time, from the worktree root:

```bash
go install golang.org/x/tools/cmd/deadcode@latest

# Stable profile
deadcode -tags '' ./cmd/camp 2>&1 | tee /tmp/deadcode-stable.txt

# Dev profile
deadcode -tags dev ./cmd/camp 2>&1 | tee /tmp/deadcode-dev.txt

# Symbols dead in BOTH profiles:
comm -12 <(sort /tmp/deadcode-stable.txt) <(sort /tmp/deadcode-dev.txt)
```

Delete only symbols present in the BOTH intersection. If a symbol appears in only one profile it means it is reachable in the other and must not be deleted.

### Step 2: Inventory to delete (verify by grep during execution)

The following inventory is from the review at `5c22d2b`/`11c270f`. Verify each cluster still exists in the worktree before deleting.

**`internal/nav` (27 symbols)**
- `internal/nav/tui/picker.go:51-172`: `Pick`, `PickScoped`, `PickPath` (the entire TUI picker; navigation moved to `go-fuzzyfinder`)
- `internal/nav/index/query.go:38-240`: entire `Query` API
- `internal/nav/builder.go:229`: `Builder.BuildWithOptions`
- `internal/nav/direct.go:127`: `JumpToPath`

Verify no references exist:
```bash
grep -rn "nav\.Pick\|tui\.Pick\|\.PickScoped\|\.PickPath\|\.JumpToPath\|\.BuildWithOptions\|nav\.Query\b" \
  --include="*.go" . | grep -v "_test.go" | grep -v "picker.go\|query.go\|builder.go\|direct.go"
```

**`internal/clone` (24 symbols)**
- `internal/clone/validation.go:55-105`: `NewValidator`, `DefaultValidator.{RegisterCheck,Validate,Checks}`
- `internal/clone/check_commits.go`: entire file
- `internal/clone/check_initialized.go`: entire file
- `internal/clone/check_urls.go`: entire file
- `internal/clone/validation_output.go:10-162`: entire file

Verify:
```bash
grep -rn "clone\.NewValidator\|clone\.DefaultValidator\|\.RegisterCheck\|clone\.ValidationCheck\|ValidationOutput\b" \
  --include="*.go" . | grep -v "_test.go" | grep -v "validation"
```

**`internal/concept/service.go:385-680` (FSService, ~295 lines)**
- `NewFSService` at `:392` and all 9 methods
- The live implementation is `DefaultService` (line 23)

Verify:
```bash
grep -rn "FSService\|NewFSService" --include="*.go" . | grep -v "_test.go" | grep -v "service.go"
```

After deletion, `internal/concept/service.go` shrinks from 680 to ~385 lines, eliminating one ARCH-6 file.

**`internal/shortcuts` (13 symbols)**
- `internal/shortcuts/expand.go:25-105`: `Expander` type and all methods
- `internal/shortcuts/feedback.go:20-206`: `FeedbackWriter` type, all methods, and the Levenshtein implementation

Verify (note: `cmd/camp/navigation/shortcuts.go` IS a live importer of `internal/shortcuts`; verify it does not use `Expander` or `FeedbackWriter`):
```bash
grep -rn "shortcuts\.Expander\|shortcuts\.FeedbackWriter\|\.FeedbackWriter\|NewExpander\b" \
  --include="*.go" . | grep -v "_test.go" | grep -v "expand.go\|feedback.go"
```

**`internal/git` (17 symbols)**
- `internal/git/branches.go:151`: `MergedBranches` (note: `MergedBranchesFromRef` at `:166` may be live; verify separately)
- `internal/git/executor.go:203-226`: `SubmoduleExecutor` and its `Info()` and `ParentNeedsCommit()` methods (verify `NewSubmoduleExecutor` at `:203` is also dead)
- `internal/git/config.go:24,35`: `GetUserEmail`, `GetUserIdentity` (also ERR-6 Get-prefix violations)
- `internal/git/errors.go:184-200`: `NewLockError`, `NewStaleLockError`, `NewActiveLockError`

Verify each:
```bash
grep -rn "git\.GetUserEmail\|git\.GetUserIdentity\|\.MergedBranches\b\|SubmoduleExecutor\|NewSubmoduleExecutor\|NewLockError\|NewStaleLockError\|NewActiveLockError" \
  --include="*.go" . | grep -v "_test.go" | grep -v "config.go\|branches.go\|executor.go\|errors.go"
```

**`internal/sync` preflights (8 symbols)**
- `internal/sync/preflight.go:121-268`: `CheckUncommittedChanges`, `checkUncommittedChanges`, `getChangesDetails`, `CheckUnpushedCommits`, `checkUnpushedCommits`, `CheckURLMismatches`, `checkURLMismatches`, `CheckDetachedHEADs`, `checkDetachedHEADs`

Note: `RunPreflight` at `:342` is a public method on `Syncer` and must be checked separately for liveness. The four exported `Check*` methods are the dead ones per ARCH-5.

Verify:
```bash
grep -rn "\.CheckUncommittedChanges\|\.CheckUnpushedCommits\|\.CheckURLMismatches\|\.CheckDetachedHEADs" \
  --include="*.go" . | grep -v "_test.go" | grep -v "preflight.go"
```

**Misc symbols**
- `internal/campaign/cache.go:68-85`: `ClearCache`, `WarmCache` -- verify neither is called outside the package
- `internal/campaign/detect.go:169-196`: `DetectWithTimeout`, `DetectFromCwdWithTimeout`, `DetectFromCwd`, `IsCampaignRoot`, `CampaignPath` -- verify each
- `internal/workflow/service_crawl.go:9`: `Service.Crawl` -- verify no caller outside the package
- `internal/workflow/transition.go:26`: `DetectTransition` -- verify no caller
- `internal/errors/errors.go:84,110,136`: `NewConfig`, `NewIO`, `NewPermission` -- verify no live caller
- `internal/errors/wrap.go:56,59`: `As`, `Unwrap` -- these wrap stdlib; verify no live caller
- `cmd/camp/dungeon/crawl.go:307`: verify the specific dead function at that line
- `cmd/camp/pins.go:281`: `pinNotFoundError` -- verify no live caller outside the function's own file

  ```bash
  grep -n "func pinNotFoundError" cmd/camp/pins.go
  ```

  The review cites it as dead at `:281`. In the worktree it may have drifted. Read the file around that line and grep for callers before deleting.

- `cmd/camp/run.go:198`: `isProjectCtx` -- verify. `isProject` at `:191` wraps it; if `isProject` is also dead, delete both.

  ```bash
  grep -rn "isProjectCtx\b\|isProject\b" cmd/camp/ --include="*.go" | grep -v "run.go"
  ```

### Step 3: `internal/config/allowlist.go:39-108` reconciliation

**Important:** sequence 06 task 02 (`06_atomicity_and_locking`) touches `allowlist.go:84` (the `os.WriteFile` that needs to become `fsutil.WriteFileAtomically`). The dead code cluster in `allowlist.go:39-108` covers `AllowlistConfig`, `LoadAllowlistConfig`, `SaveAllowlistConfig`, `DefaultAllowlistConfig`, `IsAllowed`, `AllowlistConfigExists`.

Before deleting these:
1. Check whether sequence 06 task 02 has already executed. If it has, check whether the atomic write fix landed in any of the symbols slated for deletion.
2. If sequence 06 task 02 has NOT run yet and the entire `AllowlistConfig` cluster is confirmed dead by the deadcode tool in both profiles, delete it now. The atomic write fix for `allowlist.go:84` in sequence 06 becomes a no-op because the dead code is gone.
3. If there is any ambiguity, leave the `allowlist.go` cluster for sequence 06 and exclude it from this sweep.

Record the decision in the task notes during execution.

### Step 4: `internal/crawl/fake_prompt.go` - move to test file

This is not a deletion: the 6 methods in `fake_prompt.go` are used by tests but live in a production file. Move them to `internal/crawl/fake_prompt_test.go`:

1. Read `internal/crawl/fake_prompt.go`
2. Create `internal/crawl/fake_prompt_test.go` with `package crawl_test` (or `package crawl` if the test is white-box)
3. Move the `FakePrompt` type and its methods into the new file
4. Delete `internal/crawl/fake_prompt.go`
5. Verify tests still pass: `go test ./internal/crawl/...`

### Step 5: Execute the deletion

Once every grep confirms a cluster is dead, delete it. Prefer editing files to remove the dead symbols over deleting entire files (some files like `internal/concept/service.go` have live code alongside the dead cluster). Delete entire files only when the full file content is dead.

Files that may be fully deleted if the entire file is dead:
- `internal/clone/check_commits.go`
- `internal/clone/check_initialized.go`
- `internal/clone/check_urls.go`
- `internal/clone/validation_output.go` (if lines 10-162 are the entire file)

After editing, run:
```bash
just build
BUILD_TAGS=dev just build
go test ./...
go test -tags dev ./...
```

Fix any compile errors (a caller you missed will surface here).

### Step 6: Re-run deadcode to confirm

```bash
deadcode -tags '' ./cmd/camp 2>&1 | grep -f <(cat /tmp/deadcode-stable.txt) | head -20
```

If any inventory item still appears, investigate and fix before committing.

### Step 7: Commit

One commit for the entire sweep:

```
chore(arch): dead code sweep - remove 216 dead symbols (ARCH-5, ERR-6)

Clusters removed: internal/nav TUI picker and Query API, internal/clone
validation framework, internal/concept FSService, internal/shortcuts
Expander/FeedbackWriter, internal/git MergedBranches/SubmoduleExecutor/
GetUserEmail/GetUserIdentity/NewLockError family, internal/sync
CheckUncommitted/CheckUnpushed/CheckURL/CheckDetachedHEADs,
internal/errors NewConfig/NewIO/NewPermission/As/Unwrap,
internal/campaign ClearCache/WarmCache variants,
internal/workflow Service.Crawl/DetectTransition,
internal/config AllowlistConfig cluster, misc cmd/camp dead fns.
Moved internal/crawl/fake_prompt.go to _test.go.
```

### Verified negatives (C-3 certified clean, do not re-investigate)

The second-pass adversarial review certified the following as live and must NOT be touched:
- `internal/leverage` reset/backfill boundary code
- `gendocs` glob scope
- `internal/shortcuts` reset confirmation flow (the `runShortcutsReset` function in `cmd/camp/navigation/shortcuts.go` IS live; only the `Expander`/`FeedbackWriter` in the `internal/shortcuts` package are dead)

### Edge cases

- `MergedBranches` vs `MergedBranchesFromRef`: the review cites `MergedBranches` at `:151` as dead. `MergedBranchesFromRef` at `:166` is a sibling. Verify both separately before deleting either.
- `internal/sync` preflight methods are on the `Syncer` receiver. `RunPreflight` itself is the public entry point and may be live; only the four `Check*` exported methods and their private counterparts are cited as dead.
- If the `internal/config/allowlist.go` symbols are already deleted by an earlier sequence, the deadcode output will simply not list them and nothing further is needed.

## Done When

- [x] `deadcode -tags '' ./cmd/camp` and `deadcode -tags dev ./cmd/camp` output contain none of the inventory items listed above
- [x] `just build` and `BUILD_TAGS=dev just build` pass
- [x] `go test ./...` and `go test -tags dev ./...` pass
- [x] `internal/crawl/fake_prompt.go` does not exist; a `fake_prompt_test.go` exists with the moved methods
- [x] `GetUserEmail` and `GetUserIdentity` do not appear in `internal/git/config.go`
- [x] Single sweep commit with the message above in the git log

## Execution Notes

- Sequence 06 had already run. The `internal/config/allowlist.go` cluster remained dead in both stable and dev profiles, so the whole cluster was deleted; the sequence 06 atomic-write concern for that file is now a no-op.
- The production `internal/campaign` cache helpers were deleted from the command graph. Their cache-behavior tests now use package-local `_test.go` helpers, and external tests use `CAMP_CACHE_DISABLE=1` for isolation.
- The sweep intentionally removed only the task-listed both-profile inventory, plus `JumpToPath` from the explicit nav inventory. Other deadcode output remains for separate work, and `pkg/commitkit` exports remain excluded.