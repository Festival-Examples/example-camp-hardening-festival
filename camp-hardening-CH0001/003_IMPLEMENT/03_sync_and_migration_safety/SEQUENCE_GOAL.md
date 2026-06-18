---
fest_type: sequence
fest_id: 03_sync_and_migration_safety
fest_name: sync_and_migration_safety
fest_parent: 003_IMPLEMENT
fest_order: 3
fest_status: completed
fest_created: 2026-06-11T23:26:34.397145-06:00
fest_updated: 2026-06-14T20:57:47.445642-06:00
fest_tracking: true
---


# Sequence Goal: 03_sync_and_migration_safety

**Sequence:** 03_sync_and_migration_safety | **Phase:** 003_IMPLEMENT | **Status:** Completed | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Eliminate every data-loss path in the sync, pull, migration, and intent/promote move layer so that no routine camp operation can silently destroy uncommitted content, clobber existing files, or leave the repo in an irrecoverable half-migrated state.

**Contribution to Phase Goal:** Sequence 01 greened the test gate; sequence 02 guarded destructive file-removal commands. This sequence closes the second cluster of P0 data-loss findings: the sync fallback that RemoveAlls submodule directories and replaces them with non-submodule clones (N-6), the pull loop that aborts pre-existing rebases (R7/GIT-3), every migration path that commits partial state or silently clobbers destinations (R4: INIT-1/WF-3/WF-4), and the intent moveFile paths that truncate existing files on rename errors or EXDEV (R10/WF-5, N-19, N-12). Every fix is structured as a small, extractable helper with no-replace semantics so sequence 11's shared statusdir primitive (D003) can absorb it mechanically.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Sync fallback quarantine**: `InitFromDefaultBranch` quarantine-renames non-empty dirs instead of RemoveAll, performs the fallback via real submodule machinery, resolves relative URLs against the superproject remote, and reports per-submodule outcome accurately. Container test with a hostile-remote fixture confirms the dirty submodule survives quarantine and the repo remains a real submodule.
- [ ] **Sync checkout drift**: `CheckoutDefaultBranch` errors surfaced per-submodule; `validateUpdate` correctly flags `+` drift; branch-tip != recorded-gitlink triggers a fast-forward attempt or loud warning; sync exit code reflects drift per its options table. Unit and container tests assert non-success on a stale-local-main scenario and no silent pointer regression on subsequent refs sync.
- [ ] **Pull rebase guard**: `pullSingleTarget` probes `IsRebaseInProgress` before pulling each target and skips with a warning; `handlePullError` aborts only rebases the loop initiated; container test with a pre-existing conflicted rebase confirms the conflict state is intact after `camp pull all`.
- [ ] **Repair migration all-or-nothing**: `ExecuteMigrations` pre-validates every (src exists, dst absent) pair before moving anything; mid-sequence failure stops, reports moved/unmoved, exits non-zero; `commitRepairChanges` never runs after a failed migration; mover is a small no-replace helper targeting D003 extraction. Unit tests cover pre-validation rejection, mid-failure non-commit, and clean rerun behavior.
- [ ] **Workflow migrate collision check**: `MigrateV1ToV2` destination-checks each item before renaming; per-item errors collected; on failure schema remains v1 with a clear non-zero error. Unit tests cover active/root collision and partial-failure consistent-state.
- [ ] **Intent moveFile and promote rollback**: `moveFile` has an ErrFileExists guard via O_CREATE|O_EXCL on the fallback open plus a stat guard before rename; copy fallback fires only on `errors.Is(err, syscall.EXDEV)`; promote tracks directory creation with `os.Mkdir` and removes only created paths on rollback. Unit tests cover edited-id collision, non-EXDEV rename error, and pre-existing design dir preservation.

### Quality Standards

- [ ] **Both-profile gate**: `just build`, `just test unit` (or `go test -count=1 -short ./...`), and `just lint` (or `go vet ./...`) pass in both stable and `BUILD_TAGS=dev` profiles after every task commit in this sequence.
- [ ] **No auto-commit of partial state**: `commitRepairChanges` is provably unreachable on any error path in `ExecuteMigrations`; a test asserts non-zero exit with zero commits after any failure.
- [ ] **No new host-git test files**: all destructive-path coverage for sync and pull lives in `tests/integration/`; migration and intent coverage uses host `t.TempDir()` per C-9.
- [ ] **No emdashes**: all task and sequence documents use periods, commas, or "and" instead of emdashes.
- [ ] **D003 structure**: every new mover helper is a self-contained function with a clear no-replace contract documented in the function comment, ready for extraction by sequence 11.05.

### Completion Criteria

- [ ] All six tasks in sequence completed successfully
- [ ] Both-profile gate passing after every task
- [ ] Container tests for tasks 01 and 03 passing under `just test integration`
- [ ] Unit tests for tasks 02, 04, 05, and 06 added and green
- [ ] Code review completed and issues addressed
- [ ] No `[REPLACE` or `[FILL` markers remain in any sequence document

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_sync_fallback_quarantine | Fix `InitFromDefaultBranch` to quarantine-rename non-empty dirs, use real submodule machinery, and resolve relative URLs correctly | Closes N-6: eliminates the RemoveAll-then-non-submodule-clone data-loss path; provides the hostile-remote container fixture reused by task 03 |
| 02_sync_checkout_drift_validation | Surface `CheckoutDefaultBranch` errors and correct `validateUpdate`'s backwards drift logic | Closes N-10: ensures sync exits non-zero and loud on drift; prevents silent pointer regressions on subsequent refs sync |
| 03_pull_all_rebase_guard | Probe `IsRebaseInProgress` before each pull, skip pre-existing rebases, abort only self-initiated ones | Closes R7/GIT-3: pre-existing conflict state survives `camp pull all` unmodified |
| 04_repair_migration_all_or_nothing | Pre-validate all migration pairs; all-or-nothing execution; guard `commitRepairChanges` | Closes R4/INIT-1/WF-3: partial migration can no longer be auto-committed; rerun after failure is safe |
| 05_workflow_migrate_collision_check | Add destination-exists checks and per-item error collection to `MigrateV1ToV2` | Closes R4/WF-4: v1-to-v2 migration no longer silently clobbers existing root items |
| 06_intent_movefile_and_promote_rollback | Guard `moveFile` with O_EXCL + EXDEV-only fallback; fix promote rollback to track creation with `os.Mkdir` | Closes R10/WF-5/N-19/N-12: edited-id collisions, non-EXDEV truncation, and pre-existing design dir destruction all eliminated |

## Dependencies

### Prerequisites (from other sequences)

- **01_gate_restoration**: Unit gate must be green in both profiles before this sequence; failing tests mask regressions introduced here.
- **02_destructive_command_safety**: Container test harness patterns established in sequence 02 are reused here (hostile-remote fixture for tasks 01 and 03). The sequence can begin before 02 completes but the container tests must land after the harness is in place.

### Provides (to other sequences)

- **All-or-nothing mover helpers (tasks 04, 05, 06)**: Self-contained no-replace helper functions structured for extraction into the shared statusdir primitive by sequence 11.05 (D003). The regression tests written here are the conformance baseline that 11.05 must pass.
- **Hostile-remote container fixture (task 01)**: The force-push fixture created for `InitFromDefaultBranch` testing is reused by task 03's rebase-guard container test and is available for any later sequence needing a "recorded SHA unreachable on remote" scenario.
- **Sync/pull correctness baseline (tasks 01-03)**: Sequence 06 (atomicity and locking) and sequence 07 (commit semantics) both rely on sync being structurally correct before adding write-correctness guarantees on top.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| File:line drift from review citations vs worktree at bc2ad1f | Medium | Low | Every task verifies its targets against the worktree and relocates by symbol if lines moved; drift is noted in the task Implementation section |
| macOS /var to /private/var symlink causes path-comparison failures in new tests | Medium | Medium | Use `filepath.EvalSymlinks()` on all paths before comparison; pattern is documented in the memory nuggets and already used in `internal/project/link.go:243-278` |
| EXDEV detection is platform-specific | Low | Medium | Detect via `errors.Is(err, syscall.EXDEV)` with `*os.LinkError` unwrapping; document that on Linux EXDEV surfaces identically; test on both macOS and Linux in the container |
| Quarantine-rename leaves orphaned sibling dirs if subsequent fetch also fails | Low | Medium | Document the quarantine dir in the error message so the user knows what to inspect; the quarantine is always preferable to data loss |
| D003 extraction assumption proves wrong in sequence 11 | Low | Low | Sequence 11.05 is explicitly scoped to migrate the helpers written here; if the primitive design changes, the helpers are updated in that task, not here |

## Progress Tracking

### Milestones

- [ ] **Milestone 1 (sync layer)**: Tasks 01 and 02 complete; `camp sync --force` no longer destroys non-empty submodule dirs; drift is surfaced.
- [ ] **Milestone 2 (pull and migration layer)**: Tasks 03, 04, and 05 complete; pull loop never aborts pre-existing rebases; all migration paths are all-or-nothing.
- [ ] **Milestone 3 (intent and promote layer)**: Task 06 complete; all six tasks passing both-profile gate; container and unit test suites green.

## Quality Gates

### Testing and Verification

- [ ] Container tests for tasks 01 and 03: `just test integration` passes with the new hostile-remote and rebase-guard cases
- [ ] Unit tests for tasks 02, 04, 05, and 06 pass in both stable and dev profiles
- [ ] `go test -count=1 -short ./...` and `go test -tags dev -count=1 -short ./...` both green after the final task
- [ ] No new host-git test files (the `lint-no-host-fs-tests` ratchet does not grow)

### Code Review

- [ ] Each fix reviewed for the D003 no-replace contract before merging
- [ ] EXDEV detection portability reviewed (macOS `*os.LinkError` wrapping vs Linux)
- [ ] `commitRepairChanges` gate logic verified to be unreachable on any error path

### Iteration Decision

- [x] Need another iteration? No
- [x] If yes, new tasks created: N/A
