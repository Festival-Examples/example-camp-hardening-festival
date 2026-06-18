---
fest_type: sequence
fest_id: 07_commit_semantics
fest_name: commit_semantics
fest_parent: 003_IMPLEMENT
fest_order: 7
fest_status: completed
fest_created: 2026-06-11T23:26:34.467424-06:00
fest_updated: 2026-06-14T23:51:54.963842-06:00
fest_tracking: true
---


# Sequence Goal: 07_commit_semantics

**Sequence:** 07_commit_semantics | **Phase:** 003_IMPLEMENT | **Status:** Completed | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Replace the `--only` engine with a temporary-index snapshot engine so that every scoped commit in camp captures exactly the bytes that were staged or add-time-snapshotted, then scope all auto-commit sites that currently sweep unrelated staged content.

**Contribution to Phase Goal:** The scoped-commit engine underpins every auto-commit path in camp (quest, intent, workitem, dungeon, repair) and is the public API surface fest depends on via `pkg/commitkit`. Until N-4 is fixed, the engine's core primitive commits worktree content rather than the index, and every scoped commit site (syncParentRef, SyncSubmoduleRef, refs-sync) can sweep unrelated pre-staged changes into a commit that bears a workitem or campaign tag. This sequence closes both the semantic gap and the blast-radius problem before any other sequence extends the scoped-commit pattern.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Temp-index engine**: `stageAndCommit` in `internal/git/commit/commit.go` uses `GIT_INDEX_FILE=<tmp>` + `git read-tree HEAD` + `git add -- <paths>` for auto-commit flows, and clones the real index for `--staged` flows; `--only` is removed from `executeCommit`; `git.Commit` and the lock-retry wrapper are updated to accept and propagate a temp-index env var.
- [ ] **Rename-aware staged-file list**: `listStagedFiles` in `internal/commands/workitem/staging_git.go` parses `diff --cached --name-status -z` including both sides of R/C records, matching the parser already in `ExpandTrackedPaths`.
- [ ] **Scoped sync commits**: `syncParentRef` in `cmd/camp/project/commit.go` and `SyncSubmoduleRef` in `pkg/commitkit/commitkit.go` commit scoped to the submodule path using the new engine; `HasStagedChanges` guard replaced with a path-scoped check; `refs-sync` commits with `Only: toSync` and prints skipped submodules with reasons.
- [ ] **Root commit scope and ErrNoChanges**: `camp commit` final commit is scoped to camp-staged paths or refuses pre-staged `projects/*` gitlinks without `--include-refs`; `ClassifyGitError` matches both git no-changes phrases; refs-only-drift surfaces the friendly hint.
- [ ] **Fest coordination workitem**: A design workitem exists in camp's workitem store describing `fest/internal/commands/commit/commit.go:541-553`'s unscoped reuse and the required mirror change, referencing D001 and this festival.

### Quality Standards

- [ ] **API compatibility**: `commitkit.SyncSubmoduleRef` and `commitkit.Commit` signatures are unchanged; semantic changes are documented in the package-level doc comment so fest consumers know what changed.
- [ ] **Regression test coverage**: N-4 scratch-repo case (staged v2, worktree v3, `--staged` commits v2), N-5 `git mv` case (rename commits cleanly, no stray staged D), and unrelated-staged-file survival test for sync commits all pass.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_only_index_snapshot | Replace `--only` with temp-index snapshot engine | Foundation: all other tasks build on correct commit primitives |
| 02_staged_rename_parsing | Fix `listStagedFiles` to include both rename sides | Ensures `--staged` path builds a complete scope; prevents stray staged deletions |
| 03_sync_commit_scoping | Scope syncParentRef, SyncSubmoduleRef, and refs-sync | Closes R6/GIT-2/GIT-7: pre-staged content no longer swept by sync commits |
| 04_root_commit_scope_and_errnochanges | Scope root camp commit; classify both no-changes phrases | Closes N-14 and N-23: root commit cannot sweep pre-staged gitlinks silently |
| 05_fest_commitkit_workitem | Create fest design workitem via camp tooling | Documents the fest-side mirror needed; closes correction 2 tracking requirement |

## Dependencies

### Prerequisites (from other sequences)

- Sequence 01_gate_restoration: unit gate must be green in both profiles before this sequence's regression tests are meaningful.
- Sequence 04_gate_enforcement: `just gate` and `BUILD_TAGS=dev just gate` must be available as the final gate command; until sequence 04 lands, run the standing commands from the implementation plan directly.

### Provides (to other sequences)

- Trustworthy scoped-commit engine: consumed by every subsequent sequence that touches auto-commit flows (sequences 08, 09, 11) and by fest via `pkg/commitkit`.
- Verified `ExpandTrackedPaths` and rename parser: reused in sequence 11.04 (git plumbing convergence) without re-auditing.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| GIT_INDEX_FILE env not propagated through executor chain | Med | High | Verify env propagation in `executeCommit`; add a test that sets a temp index env and confirms the env reaches the subprocess. |
| Worktree gitdir resolution breaks temp-index path | Med | High | Use `ResolveGitDir` (already in `internal/git/lock.go:89`) to locate the actual `.git/index`; add a worktree-fixture test. |
| commitkit API change breaks fest build | Med | High | Keep `SyncSubmoduleRef` and `Commit` signatures identical; run `go build ./...` from the fest submodule as a compile-compat check noted in the task. |
| Lock-retry wrapper conflicts with temp-index env | Low | Med | Pass the env through `WithLockRetry`'s closure so each retry sees the same env; verify the temp file is cleaned up on retry failure. |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: Temp-index engine passes the N-4 regression test (staged v2, worktree v3 commits v2 under `--staged`).
- [ ] **Milestone 2**: N-5 regression test and sync-scoping container test both pass; both-profile gate green.
- [ ] **Milestone 3**: Root commit scope, ErrNoChanges fix, and fest workitem complete; `just gate` green end to end.

## Quality Gates

### Testing and Verification

- [ ] `just build` green (stable profile)
- [ ] `BUILD_TAGS=dev just build` green (dev profile)
- [ ] `just test unit` green (stable)
- [ ] `go test -tags dev ./...` green (dev; or `BUILD_TAGS=dev just test unit` after sequence 04)
- [ ] `just lint` green (stable)
- [ ] `BUILD_TAGS=dev just lint` green (dev)
- [ ] `just test integration` green for any container tests added in tasks 01 and 03

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [ ] Need another iteration? No
- [ ] If yes, new tasks created: N/A
