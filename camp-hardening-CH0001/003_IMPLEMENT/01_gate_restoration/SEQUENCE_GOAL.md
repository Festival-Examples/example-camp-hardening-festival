---
fest_type: sequence
fest_id: 01_gate_restoration
fest_name: gate_restoration
fest_parent: 003_IMPLEMENT
fest_order: 1
fest_status: completed
fest_created: 2026-06-11T23:26:34.355932-06:00
fest_updated: 2026-06-12T13:26:43.793268-06:00
fest_tracking: true
---


# Sequence Goal: 01_gate_restoration

**Sequence:** 01_gate_restoration | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Make the unit test gate green in both build profiles (stable and dev) so every subsequent sequence has a meaningful, non-red quality gate to close against.

**Contribution to Phase Goal:** This sequence is a bootstrap precondition, not gate-enforcement work in the substantive sense. The full test suite is currently red on main (TEST-1). Without this fix, every later sequence's auto-appended testing gate either fails on the pre-existing manifest failure or is waved through over a red baseline, normalizing exactly the gate-rot the festival exists to fix. Once green, all 02-12 sequences gain a trustworthy baseline to measure regressions against. Recorded by decision D007.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Manifest allowlist fix**: `"workitem priority": true` present in the `agentAllowed` map in `TestManifestCommand_AllCommandsHaveAnnotations`, and `"workitem priority": false` present in the `expectedCommands` workitem block in `TestManifestCommand_AllRestrictedCommandsPresent`, with the workitem count increment changed from 9 to 10 (`cmd/camp/manifest_test.go`).
- [ ] **Both-profile gate green**: `go test -count=1 -short -run TestManifestCommand ./cmd/camp` passes in stable (no tags) and under `-tags dev`; the full `go test -count=1 -short ./...` run completes without new failures in stable; `go test -tags dev -count=1 -short ./...` completes without new failures in dev.
- [ ] **Baseline survey recorded**: a `BASELINE.md` file in this sequence directory capturing pass/fail for every command in the standing verification matrix (`just build`, `BUILD_TAGS=dev just build`, `just test unit`, `go test -tags dev -count=1 -short ./...`, `just lint`, `go vet -tags dev ./...`, `go vet -tags=integration ./...`) with owning-sequence noted for each failure. No fixes beyond R1.

### Quality Standards

- [ ] **Scope discipline**: the only code change in this sequence is the manifest test file. Any other red items found during the baseline survey are recorded with their owning sequence number, not fixed here (D007 scope cap).
- [ ] **Both profiles verified**: every test command is run in both stable and dev profiles; a result from only one profile does not satisfy the gate.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_fix_manifest_allowlist | Add `workitem priority` to the manifest test allowlist and bump expected restricted-command counts | Turns the red TEST-1 gate green in both profiles; satisfies the R1 deliverable |
| 02_baseline_gate_survey | Run the full standing verification matrix, record pass/fail per command in BASELINE.md | Documents the starting health of every gate so sequences 02-12 can track regressions cleanly |

## Dependencies

### Prerequisites (from other sequences)

- None. This is the first sequence. The worktree at `projects/worktrees/camp/camp-hardening` (branch point `bc2ad1f`) is the starting state.

### Provides (to other sequences)

- Green both-profile unit baseline: used by sequences 02 through 12 as the required pre-condition for meaningful quality gates on every implementation task.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Additional manifest test failures found during baseline reveal a deeper annotation gap | Low | Medium | Record in BASELINE.md for task 09_app_contracts (sequence 09); do not fix in this sequence per D007 scope cap |
| Baseline survey reveals dev-profile compile failures unrelated to R1 | Low | Medium | Record with owning sequence; do not fix; the sequence still closes once R1 is green in both profiles |
| Line drift between review citations and the worktree | Low | Low | Task 01 instructs executor to verify file:line targets before editing; worktree is pinned at bc2ad1f |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: `TestManifestCommand` green in stable profile
- [ ] **Milestone 2**: `TestManifestCommand` green in dev profile
- [ ] **Milestone 3**: BASELINE.md committed with full standing matrix results

## Quality Gates

### Testing and Verification

- [ ] `go test -count=1 -short -run TestManifestCommand ./cmd/camp` passes (stable)
- [ ] `go test -tags dev -count=1 -short -run TestManifestCommand ./cmd/camp` passes (dev)
- [ ] `go test -count=1 -short ./...` produces no new failures beyond those recorded in BASELINE.md
- [ ] Integration tests: not required for this sequence (no container tests added)

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [ ] Need another iteration? No
- [ ] If yes, new tasks created: none yet