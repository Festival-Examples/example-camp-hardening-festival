---
fest_type: sequence
fest_id: 04_gate_enforcement
fest_name: gate_enforcement
fest_parent: 003_IMPLEMENT
fest_order: 4
fest_status: completed
fest_created: 2026-06-11T23:26:34.415205-06:00
fest_updated: 2026-06-14T21:42:28.596014-06:00
fest_tracking: true
---


# Sequence Goal: 04_gate_enforcement

**Sequence:** 04_gate_enforcement | **Phase:** 003_IMPLEMENT | **Status:** Completed | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Establish trustworthy local quality gates in both build profiles (stable and dev) so every subsequent sequence verifies against a gate that actually runs all code, including dev-tagged code that festival-app ships; no GitHub Actions or hosted CI is involved anywhere (C-1).

**Contribution to Phase Goal:** Sequences 01-03 restore the baseline and fix data-loss paths. This sequence is the gate-infrastructure delivery: after it lands, `just gate` and `just gate-fast` are the single authoritative commands every later sequence closes against; the pre-push hook makes it impossible to silently merge a red tree; and the five dead integration-tagged host test files and the stale container Reset can no longer produce phantom coverage or latent flakes. Sequences 05-12 and the POLISH phase all depend on the commands this sequence creates.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **tasks.Test honors BUILD_TAGS**: `internal/buildutil/tasks/test.go` reads `BUILD_TAGS` exactly as `tasks.Build` does; dev-tagged tests in `cmd/camp/quest/`, `cmd/camp/release_profile_dev_test.go`, and `internal/quest/tui/` run when `BUILD_TAGS=dev just test unit` is invoked.
- [ ] **just gate-fast and just gate recipes**: both recipes exist in the justfile following D005 exactly (gate-fast: both-profile builds, vet with stable/dev/integration tag sets, lint both profiles, dev-profile unit tests; gate: gate-fast plus stable-profile unit tests).
- [ ] **Release recipes gate on just gate**: every recipe in `.justfiles/release.just` (dev, rc, stable, promote-rc, promote) invokes `just gate` before tagging; the festival-repo division of labor is documented in the recipe file.
- [ ] **Committed .githooks/pre-push hook**: the hook file exists, runs `just gate-fast` (or `just gate` when `CAMP_GATE_FULL=1`), prints a clear failure banner naming the reproduce command, and satisfies the githooks(5) stdin contract. `just hooks install` sets `core.hooksPath .githooks` idempotently. No `.github/workflows/` files anywhere in the diff.
- [ ] **Five dead integration-tagged host test files resolved**: `cmd/camp/commit_integration_test.go`, `cmd/camp/pull_integration_test.go`, `cmd/camp/status_integration_test.go`, `cmd/camp/refs/commands_integration_test.go`, and `internal/git/lock_integration_test.go` are either deleted or have their worthwhile coverage migrated into `tests/integration/` following the container harness conventions.
- [ ] **go vet -tags=integration ./... clean and wired**: the vet command passes clean against the worktree and is invoked inside `just gate-fast` (which `just lint` already calls via `gate-fast`).
- [ ] **Container Reset fixed**: `tests/integration/helpers_container.go` Reset removes `/root/.obey` and per-test fixed work roots; legacy `/root/.config/camp` entry is dropped; the manual `rm -rf /root/.obey/campaign` in `tests/integration/settings_test.go:16` is removed.

### Quality Standards

- [ ] **Both profiles, every gate**: every gate command runs in both stable and dev profiles per C-2; a result from one profile alone does not satisfy any verification check.
- [ ] **No hosted CI**: zero `.github/workflows/` files added or touched (C-1); gate enforcement is entirely `just` recipes plus the committed hook.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_test_runner_build_tags | Make `tasks.Test` honor `BUILD_TAGS`; add `just gate-fast` and `just gate` | Delivers the both-profile test runner and the two gate recipes that every later sequence closes against |
| 02_release_recipe_gating | Wire `just gate` into every release recipe before tagging | Closes BR-2: no release can ship from an ungated tree |
| 03_prepush_hook | Commit `.githooks/pre-push`; add `just hooks install` | Stops silent red-gate merges at push time; keeps enforcement local per C-1 and D005 |
| 04_integration_tagged_test_migration | Delete or migrate the five dead `//go:build integration` host files; add `go vet -tags=integration ./...` to lint | Removes the illusion of commit/pull/refs coverage and closes the compile-broken tagged file that poisons any future `-tags=integration ./...` run |
| 05_container_reset_paths | Fix container pool Reset to clean the paths tests actually use | Eliminates latent flakes from inherited state; removes the manual workaround in settings_test.go |

## Dependencies

### Prerequisites (from other sequences)

- Sequence 01 (01_gate_restoration): the unit gate must be green in both profiles before gate-enforcement recipes are meaningful; D007 records this ordering.
- Sequence 02 (02_destructive_command_safety): data-loss fixes land before gate work per the FESTIVAL_GOAL ordering and C-8.
- Sequence 03 (03_sync_and_migration_safety): same data-loss ordering; `go vet -tags=integration ./...` in task 04 of this sequence will emit failures from the dead compile-broken files until they are removed, so the vet addition lands with or after the deletion in task 04 (see task 01 ordering note).

### Provides (to other sequences)

- `just gate` and `just gate-fast`: the authoritative closing command for sequences 05-12 and the POLISH phase.
- Pre-push hook: passive continuous enforcement for all contributors from this point forward.
- Clean integration vet: sequences that add container tests can now include `go vet -tags=integration ./...` in their local verification without hitting phantom compile errors.
- Fixed container Reset: sequences 05-12 that add integration tests get a clean container between runs without manual workarounds.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| `go vet -tags=integration ./...` fails on the dead files until task 04 deletes them; adding it to gate-fast in task 01 causes a chicken-and-egg | Medium | Medium | Task 01 adds the vet line to gate-fast; gate-fast is only added to the hook in task 03 and wired to lint in task 01. The worktree's current vet already fails on integration tags, so task 04 must land before gate-fast is used as the hook body; the hook (task 03) commits the file but contributors run `just hooks install` after the sequence is complete. Implementer should run tasks 01, 04, 03 in order, or verify vet passes before wiring the hook. |
| Coverage worth migrating from the five dead files is harder to port than expected | Medium | Low | The container harness in `tests/integration/` already covers commit and pull paths; task 04 identifies which tests are net-new vs duplicated; the decision is migrate-vs-delete, not migrate-everything. |
| Line drift between review citations and the worktree at bc2ad1f | Low | Low | Every task instructs the executor to verify file:line before editing; symbol names are stable even if line numbers shift. |
| Hook bypassed with --no-verify during time pressure | Low | Medium | The clear failure banner names `just gate` as the reproduce command; `just gate` is required by release recipes independently of the hook. |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: `BUILD_TAGS=dev just test unit` runs dev-tagged tests and `just gate` / `just gate-fast` recipes exist.
- [ ] **Milestone 2**: All release recipes run `just gate` before tagging; pre-push hook committed and `just hooks install` recipe present.
- [ ] **Milestone 3**: Five dead integration-tagged files deleted or migrated; `go vet -tags=integration ./...` clean; container Reset fixed; `just gate` green end to end.

## Quality Gates

### Testing and Verification

- [ ] `BUILD_TAGS=dev just test unit` executes dev-tagged tests (confirmed by seeing `cmd/camp/quest/` and `release_profile_dev_test.go` tests in output)
- [ ] `just gate` exits 0 in the worktree after all five tasks land
- [ ] `just gate-fast` exits 0
- [ ] `go vet -tags=integration ./...` exits 0 and is wired into `just gate-fast`
- [ ] Zero `.github/workflows/` files in `git diff bc2ad1f..HEAD --stat`
- [ ] Hook fires and blocks on a deliberate failure in a scratch-push test (task 03 scripted verification)
- [ ] Container integration suite runs green twice back-to-back without manual Reset workarounds (task 05 acceptance)

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [ ] Need another iteration? No
- [ ] If yes, new tasks created: none yet
