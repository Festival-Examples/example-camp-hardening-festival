---
fest_type: sequence
fest_id: 06_atomicity_and_locking
fest_name: atomicity_and_locking
fest_parent: 003_IMPLEMENT
fest_order: 6
fest_status: completed
fest_created: 2026-06-11T23:26:34.450275-06:00
fest_updated: 2026-06-14T23:04:00.741188-06:00
fest_tracking: true
---


# Sequence Goal: 06_atomicity_and_locking

**Sequence:** 06_atomicity_and_locking | **Phase:** 003_IMPLEMENT | **Status:** Completed | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Route every state-file write through `fsutil.WriteFileAtomically`, lock every read-modify-write cycle that can race between concurrent camp invocations, and eliminate the private `atomicWriteFile` clone in `internal/commands/workitem/create.go`.

**Contribution to Phase Goal:** Festival-app polls camp stores continuously and agent loops issue parallel camp invocations. Until atomic writes and file locks cover every mutable store, a crash or concurrent invocation silently truncates an intent, corrupts a priority assignment, or loses a registry entry. This sequence makes the durability contract uniform across the codebase so every downstream sequence and the app can rely on the stores being consistent.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [x] **Intent and quest atomic writes (task 01)**: All `os.WriteFile` calls in `internal/intent/service.go`, `internal/quest/quest.go`, and any same-scope intent update helpers are converted to `fsutil.WriteFileAtomically`; file modes preserved; read-modify-write callers flagged for sequence-06 locking where non-trivial.
- [x] **Workflow, flow, state, config, and scaffold atomic writes (task 02)**: All `os.WriteFile` calls in `internal/workflow/service_init.go`, `internal/workflow/service_sync_migrate.go`, `internal/flow/registry.go`, `internal/state/location.go` (os.Create+write loop replaced), `internal/scaffold/repair.go` gitignore append, and `internal/config/allowlist.go` converted; `LoadHistory` skips corrupt lines with a warning; private `atomicWriteFile` clone in `create.go` deleted and its caller switched to `fsutil`.
- [x] **Registry lock (task 03)**: `config.UpdateRegistry(ctx, mutate func(*Registry) error)` introduced; acquires `fsutil.AcquireFileLock`, re-loads inside the lock, calls mutate, saves atomically; all mutating callers (`register.go`, `clone/register.go`, `scaffold/init.go`, `switch.go`) migrated.
- [x] **Priority store lock (task 04)**: `priority.WithLock(ctx, storePath, fn)` introduced mirroring `links.WithLock`; CLI command and TUI mutation paths migrated; `priority.Prune` remains outside the lock (moves to sequence 08 task 03).
- [x] **fsutil stale-lock TOCTOU fixed (task 05)**: `removeStaleLock` replaced with a rename-steal or PID-liveness approach so exactly one concurrent stealer wins; `staleLockAfter` made a named constant with documented rationale; behavior fail-safe (stat errors treated as active lock).

### Quality Standards

- [x] **Both-profile gate**: `just build`, `just test unit`, and `just lint` pass in both stable and `BUILD_TAGS=dev` profiles after every task.
- [x] **Grep checkpoint**: After task 02, `grep -rn "os.WriteFile" internal/intent internal/quest internal/workflow internal/flow internal/state internal/config internal/scaffold internal/commands/workitem` shows only the expected-remainder list recorded in task 02's "Done When" section.

### Completion Criteria

- [x] All tasks in sequence completed successfully
- [x] Quality verification tasks passed
- [x] Code review completed and issues addressed
- [x] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_intent_quest_atomic_writes | Convert all os.WriteFile sites in internal/intent/service.go and internal/quest/quest.go to fsutil.WriteFileAtomically | Eliminates crash-unsafe plain writes for the most frequently mutated user state files |
| 02_workflow_flow_state_config_atomic_writes | Convert workflow, flow, state, config allowlist, gitignore-append, and workitem private-clone sites; fix LoadHistory; delete clone | Completes the atomic-write sweep and removes the weaker private clone that duplicated fsutil logic without fsync |
| 03_registry_update_lock | Introduce config.UpdateRegistry with file-lock and in-lock reload; migrate all mutating callers | Prevents concurrent camp invocations from silently dropping registry entries written in the same time window |
| 04_priority_store_lock | Introduce priority.WithLock mirroring links.WithLock; migrate CLI and TUI mutation paths | Prevents concurrent priority set/clear from losing one caller's write, which festival-app depends on |
| 05_fsutil_stale_lock_toctou | Fix stat-then-remove race in removeStaleLock with rename-steal or PID-liveness | Closes the lock-infrastructure race that any of the locks added in tasks 03-04 would inherit |

## Dependencies

### Prerequisites (from other sequences)

- **01_gate_restoration**: Unit gate must be green in both profiles so the both-profile gate run after each task is meaningful.
- **04_gate_enforcement**: `just gate` and `just test unit` must work in both profiles. Until sequence 04 lands, the standing verification command is `go test -tags dev -count=1 -short ./...` for the dev profile.

### Provides (to other sequences)

- **Durable state writes**: Used by sequences 07-12; every sequence that modifies camp state relies on the atomic-write guarantee landing here first.
- **Locked registry and priority stores**: Festival-app (sequences 09 onward) polls the priority store and shells camp commands that touch the registry; correctness under concurrent access is a prerequisite for those sequences.
- **Fixed lock infrastructure**: The stale-lock TOCTOU fix in task 05 benefits any future `AcquireFileLock` caller, including the nav-index lock in sequence 10 task 06 (NAV-3).

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| File-mode regression on macOS (WriteFileAtomically preserves mode via Stat; first-create path uses defaultMode) | Low | Low | Verify 0644 is the correct default at all new call sites; add a mode assertion in tests where the mode matters |
| Rename-steal fix in task 05 introduces a new temp-file path that itself races on a slow fs | Low | Medium | Use os.MkdirTemp semantics (random suffix) for the steal-rename target; document the narrowed-but-not-eliminated window |
| UpdateRegistry callers that relied on LoadRegistry returning a cached pointer now reload from disk more often | Low | Low | The extra disk reads are negligible compared to git operations; no performance-sensitive hot path calls register repeatedly |
| TUI model holds an in-memory priorityStore; WithLock wraps disk I/O but the in-memory store can still be stale if another process mutates between TUI refreshes | Medium | Low | This is pre-existing behavior; task 04 closes the disk-write race, not the in-memory stale-read window; document the residual |

## Progress Tracking

### Milestones

- [x] **Milestone 1**: Tasks 01 and 02 complete; atomic-write sweep done; grep checkpoint passes; both-profile gate green.
- [x] **Milestone 2**: Tasks 03 and 04 complete; registry and priority stores locked; concurrency regression tests pass.
- [x] **Milestone 3**: Task 05 complete; lock infrastructure race closed; full sequence both-profile gate green.

## Quality Gates

### Testing and Verification

- [x] All unit tests pass in both profiles after every task
- [x] Grep checkpoint for os.WriteFile remainder recorded and verified (task 02)
- [x] Concurrency regression tests for registry (task 03) and priority store (task 04) pass
- [x] Stale-lock steal race test passes (task 05)

### Code Review

- [x] Code review conducted
- [x] Review feedback addressed
- [x] Standards compliance verified

### Iteration Decision

- [x] Need another iteration? No
- [x] If yes, new tasks created: N/A
