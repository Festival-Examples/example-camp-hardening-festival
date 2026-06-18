---
fest_type: sequence
fest_id: 11_debt_burn_down
fest_name: debt_burn_down
fest_parent: 003_IMPLEMENT
fest_order: 11
fest_status: completed
fest_created: 2026-06-11T23:26:34.537143-06:00
fest_updated: 2026-06-15T12:33:44.725978-06:00
fest_tracking: true
---


# Sequence Goal: 11_debt_burn_down

**Sequence:** 11_debt_burn_down | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Remove the structural debt clusters in safe dependency order (ratchet, dead code, extraction, plumbing, move-primitive unification, subsystem hardening) so that the codebase entering sequence 12 is lean, correctly layered, and hardened against the correctness bugs identified in the review.

**Contribution to Phase Goal:** This sequence closes R24, R29, R30, R31, R25, R32, R33, N-20, N-22, N-26, N-27. It eliminates ~216 dead symbols, moves business logic out of `cmd/`, fixes porcelain/plumbing correctness in the git layer, unifies the four divergent status-dir move implementations, hardens quests and the intent parser, and fixes three small safety issues. Together these reduce regression surface for sequence 12's polish pass and provide the lean codebase that makes sequence 12 tractable.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **fmt.Errorf ratchet**: `just lint-no-fmt-errorf` recipe active and wired into `just lint`; `internal/git/query.go` and `pkg/commitkit` migrated to `camperrors`; allowlist file committed
- [ ] **Dead code eliminated**: ~216 symbols dead in both profiles deleted; `deadcode` tool re-run confirms inventory gone; `internal/crawl/fake_prompt.go` methods moved to `_test.go`
- [ ] **Cmd logic extracted**: Tree B constructor style documented; pull, status_all, shortcuts, go.go resolution, leverage scoring, dungeon crawl staging, EnsureRefForCommit, and run.go duplication extracted into `internal/` in extraction-only commits
- [ ] **Git plumbing converged**: porcelain `-z` parser in use; FilterTracked uses `-z`; IsSubmodule distinguishes worktrees; ERR-8 LC_ALL=C and submodule classification tightened; GIT-10 small fixes applied; `internal/git` package converged onto RunGitCmd
- [ ] **Shared statusdir primitive**: `internal/statusmove` package with no-replace semantics (darwin and linux variants); all four move implementations migrated; dungeon TOCTOU and stat-conflation fixed; collision tests added
- [ ] **Quest robustness**: List skips-and-warns on bad entries; slug reservation via os.Mkdir; ID uniqueness checked; stable build quest surfaces channel-aware
- [ ] **Intent parser hardened**: line-anchored delimiters; unknown-key round-trip; O_EXCL on guarded writes; removeAllCopies verifies id; gather orphan cleanup and archive failure collection
- [ ] **Small safety fixes**: skills --force unmanaged symlink warning; pins migration resolves symlinked root; buildutil clean cwd guard

### Quality Standards

- [ ] **Both-profile gate after every task**: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint` all green after each individual task commit (this sequence has the highest regression surface in the festival)
- [ ] **Extraction-only commits for task 03**: no behavior changes inside any extraction commit; existing tests unchanged

### Completion Criteria

- [ ] All eight tasks completed successfully
- [ ] `deadcode -tags '' ./cmd/camp` and `deadcode -tags dev ./cmd/camp` confirm the ARCH-5 inventory is gone
- [ ] `just lint` includes `lint-no-fmt-errorf` and exits 0 in both profiles
- [ ] Both-profile gate green end to end

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_fmt_errorf_ratchet | Add grep ratchet for `fmt.Errorf`; migrate query.go and commitkit | Establishes the enforcement mechanism; removes untyped errors from the two widest-propagation sites |
| 02_dead_code_sweep | Delete ~216 dead symbols in both profiles (STRICTLY BEFORE task 03) | Clears the field for task 03; ensures no dead code gets migrated into new internal packages |
| 03_cmd_logic_extraction | Document Tree B; extract pull/status/shortcuts/leverage/dungeon/go.go into internal/ | Reduces cmd/ bloat, enables unit testing of domain logic, addresses ARCH-1/2/6/7 |
| 04_git_plumbing_convergence | Fix porcelain, FilterTracked, IsSubmodule, ERR-8, and GIT-10 small fixes | Eliminates correctness bugs in the git layer that affect status_all, commit scope, and worktree detection |
| 05_statusdir_move_primitive | Build shared move package; migrate four implementations | Eliminates TOCTOU and stat-conflation; unifies divergent collision policies; provides the R25 primitive |
| 06_quest_robustness | Fix List hard-fail, slug TOCTOU, ID uniqueness, BR-5 channel surface | Hardens quest subsystem so a corrupt file cannot brick all quest commands |
| 07_intent_parser_hardening | Fix delimiter anchoring, unknown keys, O_EXCL, removeAllCopies, gather | Closes the four medium-severity intent correctness bugs and the N-26 orphan issue |
| 08_small_safety_fixes | Fix skills symlink warning, pins symlinked root, buildutil clean guard | Three focused safety improvements with minimal risk |

## Dependencies

### Prerequisites (from other sequences)

- **Sequence 01 (gate_restoration)**: unit gate green in both profiles before any of this sequence's work is trustworthy
- **Sequence 03 (sync_and_migration_safety)**: interim no-replace helpers in `internal/intent/helpers.go` and `internal/scaffold/repair.go` that task 05 migrates and deletes (D003)
- **Sequence 04 (gate_enforcement)**: both-profile test and lint infrastructure used by every task in this sequence
- **Sequence 06 (atomicity_and_locking)**: atomic quest writes prerequisite for task 06; `allowlist.go:84` atomic write (N-21) may affect what is dead in task 02
- **Sequence 09 (app_contracts)**: `Profile` constant introduced for task 06's BR-5 channel-aware approach

### Ordering note

**Task 02 STRICTLY before task 03.** This is load-bearing (R29): the dead code sweep must complete and commit before any extraction begins so no dead code is migrated into internal packages.

### Provides (to other sequences)

- **Lean codebase for sequence 12**: after task 03's extractions the `cmd/` files shrink toward the 500-line standard; sequence 12's polish and test hygiene pass works on a smaller, cleaner surface
- **POLISH verification surface**: the 004_POLISH phase's verification checklist (D006) includes confirming the dead-code inventory is gone and the both-profile gate is green; this sequence is what makes those checks meaningful

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

Branch point: `bc2ad1f` (includes PR #318 workitem priority). Review citations are from `5c22d2b`/`11c270f`; verify every file:line target in the worktree before editing since the two commits past baseline may have drifted some line numbers.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Task 03 extraction introduces a subtle behavior change | Med | Med | One extraction per commit; run the full test suite after each commit before proceeding |
| Dead code grep misses a live caller (task 02 deletes something live) | Low | High | `deadcode` tool run in both profiles before deletion; compile errors catch missed callers immediately |
| Task 05 `statusmove` primitive breaks sequence-03 regression tests | Low | Med | Sequence-03 tests are the conformance suite; run them explicitly during task 05 |
| Task 06 channel-aware quest scaffolding breaks the Profile constant integration | Low | Med | If Profile is not available from sequence 09, use build-tag approach and note for sequence 09 cleanup |
| Line number drift between review citations and worktree bc2ad1f | Med | Low | Every task explicitly verifies targets before editing; small drift (1-5 lines) is expected and normal |

## Progress Tracking

### Milestones

- [ ] **Milestone 1 (ratchet + dead code)**: Tasks 01 and 02 complete; ratchet active in lint, dead code gone, both-profile gate green
- [ ] **Milestone 2 (extraction + plumbing)**: Tasks 03 and 04 complete; cmd/ files measurably smaller, git plumbing corrected, extraction-only commits verified
- [ ] **Milestone 3 (unification + hardening)**: Tasks 05-08 complete; shared statusmove primitive live, quest and intent subsystems hardened, small safety fixes in place; both-profile gate green end to end

## Quality Gates

### Testing and Verification

- [ ] Both-profile gate (`just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`) green after EVERY task in this sequence
- [ ] Named tests per task pass: porcelain leading-space, non-ASCII path filter, worktree-vs-submodule fixture, locale classification, corrupt quest skip, concurrent quest create, gather orphan cleanup, skills symlink warning, pins symlinked root, buildutil clean guard
- [ ] `deadcode` tool confirms inventory gone (tasks 02 verification)
- [ ] `just test integration` green for tasks that add container tests (task 03 extraction is unit-only)

### Code Review

- [ ] Extraction commits in task 03 reviewed to confirm zero behavior change
- [ ] Task 05 `statusmove` implementation reviewed for platform correctness (darwin link+unlink semantics)
- [ ] Standards compliance: no new `fmt.Errorf` outside allowlist, no Get-prefix exports added

### Iteration Decision

- [ ] Need another iteration? Determined at sequence close based on gate results
- [ ] If yes, new tasks created with specific regression identifiers