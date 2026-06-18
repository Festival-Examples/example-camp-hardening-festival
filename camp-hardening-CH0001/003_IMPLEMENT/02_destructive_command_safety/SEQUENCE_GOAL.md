---
fest_type: sequence
fest_id: 02_destructive_command_safety
fest_name: destructive_command_safety
fest_parent: 003_IMPLEMENT
fest_order: 2
fest_status: completed
fest_created: 2026-06-11T23:26:34.378467-06:00
fest_updated: 2026-06-13T12:38:08.281411-06:00
fest_tracking: true
---


# Sequence Goal: 02_destructive_command_safety

**Sequence:** 02_destructive_command_safety | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Ensure no camp command that deletes files can destroy uncommitted or untracked user work without explicit consent, by adding guards, confirmation prompts, and dirty-tree checks to the five identified destructive paths: worktrees clean, cp/copy, fresh/prune branch-worktree removal, the prune engine itself (which has zero tests), and project remove.

**Contribution to Phase Goal:** Phase 003_IMPLEMENT produces a fully hardened camp binary. This sequence eliminates the data-loss class of defects (R2+correction1, N-1, N-2, R16, N-3), which the plan places first because code that can silently destroy data must be fixed before any subsequent gate or contract work can be trusted.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **worktrees clean guards (task 01)**: `camp worktrees clean` only calls `os.RemoveAll` when the staleness reason is "gitdir target does not exist"; entries whose `.git` is a directory are listed as skipped with guidance; a confirmation prompt gates all deletions unless `--yes`/`--force` is passed or stdin is not a terminal; the empty-projectPath bypass is closed; `--force` still runs a dirty-tree check; container integration tests cover all five cases.
- [ ] **cp copy guards (task 02)**: `camp cp --force` detects `os.SameFile` and refuses instead of truncating the source; directory copies where the destination resolves under the source are rejected with cp(1)-style messaging; the non-force error message no longer steers users into the destructive `--force` invocation; container tests cover both failure modes and verify normal copies still work.
- [ ] **prune dirty-worktree protection (task 03)**: the branch-worktree removal path in `deleteLocalBranches` runs the same `detachedWorktreeClean` check that detached-worktree removal already uses; dirty branch worktrees are skipped with `SkipReasonDirtyWorktree`; an explicit `--discard-dirty` flag is required to destroy dirty worktrees; `camp fresh` stops passing unconditional `Force: true` past the dirty check.
- [ ] **prune test suite (task 04)**: `internal/prune` has table-driven unit tests covering candidate/forced/merged classification using a fake executor seam; a container integration test covers `camp project prune` including `--remote-delete` against a bare origin fixture; merged branches are pruned, unmerged are skipped, dirty worktrees are skipped, remote deletion is confirmed, dry-run is verified.
- [ ] **project remove unpushed-refs guard (task 05)**: `internal/project/remove.go` refuses to delete `.git/modules/projects/<name>` when the module gitdir contains refs not reachable from any remote unless `--force` is passed and the unreachable branches are named in the refusal message; the help text no longer falsely claims files remain in place; container tests verify refuse/force/clean paths.

### Quality Standards

- [ ] **Both-profile gate passes**: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `go test -tags dev ./...`, `just lint`, `BUILD_TAGS=dev just lint`, and `just test integration` all pass from the worktree root (`projects/worktrees/camp/camp-hardening`) after each task.
- [ ] **No GitHub Actions files touched**: the diff contains zero `.github/workflows/` additions or modifications per constraint C-1.
- [ ] **No verified-negatives re-opened**: worktree add/remove plumbing and project add/new creation guards are confirmed clean by the second pass and must not be modified (constraint C-3).
- [ ] **Destructive-path tests in the right tier**: tests touching real filesystems or git operations use `tests/integration/` with the testcontainers harness; new unit tests follow the host `t.TempDir()` convention; the `lint-no-host-fs-tests` ratchet does not grow.
- [ ] **Behavior changes use flags, not silent swaps**: new safe behavior is the default; destruction requires explicit flags per constraint C-11; help text is updated in the same change as the guard.

### Completion Criteria

- [ ] All five tasks completed successfully
- [ ] Both-profile gate passes end-to-end after all tasks
- [ ] `just test integration` passes with the new worktrees/cp/prune/remove container tests
- [ ] Manual `--dry-run` spot checks confirmed per task
- [ ] Documentation (help text, Long descriptions) updated where cited bugs included false claims

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_worktrees_clean_guards | Add confirmation prompt, restrict RemoveAll to gitdir-missing staleness only, close empty-projectPath bypass, require dirty check with --force | Closes R2/GIT-1 and correction-1; eliminates the most acute RemoveAll data-loss path |
| 02_cp_copy_guards | Add os.SameFile check and dest-under-src rejection to camp cp | Closes N-1; prevents source truncation and self-recursion on a commonly-used command |
| 03_prune_dirty_worktree_protection | Apply dirty-worktree check to branch-worktree removal; add --discard-dirty flag; fix fresh Force passthrough | Closes N-2; extends the guard that already exists for detached worktrees to branch worktrees |
| 04_prune_test_suite | Add unit tests (fake executor) and container tests for internal/prune | Closes R16/TEST-4; 618-line zero-test package that deletes remote branches gets real coverage |
| 05_project_remove_unpushed_refs_guard | Detect unpushed refs before removing .git/modules; require --force and name the branches | Closes N-3; prevents permanent loss of committed-but-unpushed history on project removal |

## Dependencies

### Prerequisites (from other sequences)

- **01_gate_restoration**: unit gate must be green in both profiles before this sequence executes; without it the passing/failing signal from the both-profile gate is unreliable.

### Provides (to other sequences)

- **Guarded destructive commands**: sequence 03_sync_and_migration_safety builds on this sequence's container-test patterns and the established `--discard-dirty` / confirmation-prompt idiom.
- **Container test patterns for worktrees and prune**: sequences that add prune or fresh tests can reference the fixtures created here (bare origin, dirty worktree setup).

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

All edits and test runs happen in this worktree. Do not modify `projects/camp` or any other path.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| File:line drift from review citations (worktree is at bc2ad1f, reviews at 5c22d2b/11c270f) | Medium | Low | Each task documents verified line numbers from the worktree; implementer re-locates by symbol if lines shift further |
| Confirmation-prompt change breaks non-TTY scripts | Medium | High | Non-TTY behavior is explicitly specified: refuse without --yes/--force when stdin is not a terminal; container tests exercise the non-TTY path |
| prune test seam introduction changes behavior | Low | High | Seam is a minimal interface injection; task 04 specifies that existing behavior must be unchanged and the unchanged test suite proves it |
| macOS /private/var symlinks break SameFile check | Medium | Medium | Task 02 explicitly calls out EvalSymlinks requirement for path comparison; container tests run in Linux so macOS edge is documented as an edge case to verify manually |
| --discard-dirty flag naming conflicts with existing flags | Low | Low | Task 03 checks existing prune.Options fields before introducing the flag; naming follows GAP-8 guidance for consistency across fresh and prune |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: Tasks 01 and 02 complete - file-level destructive commands guarded (worktrees clean, cp)
- [ ] **Milestone 2**: Tasks 03 and 04 complete - prune engine guarded and tested
- [ ] **Milestone 3**: Task 05 complete and both-profile gate passes end-to-end

## Quality Gates

### Testing and Verification

- [ ] `just build` and `BUILD_TAGS=dev just build` green in worktree
- [ ] `just test unit` and `go test -tags dev ./...` green in worktree
- [ ] `just test integration` green with new container tests
- [ ] `--dry-run` produces expected output for each changed command

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified (no fmt.Errorf, context propagation, no magic strings)

### Iteration Decision

- [ ] Need another iteration? No
- [ ] If yes, new tasks created: N/A