# Festival Overview: camp-hardening

## Problem Statement

**Current State:** The staff-level review at `workflow/explore/fable-review-2026-06-10/camp/` (first pass at commit 5c22d2b, adversarial second pass at 11c270f, 20 of 20 re-checked findings confirmed and none refuted) found camp in an uneven state: 3 critical, ~24 high, ~45 medium, and ~60 low findings. The unit test gate is red on main in both build profiles (manifest allowlist missing `workitem priority`). Several daily-use commands can destroy user or fest-owned data: `worktrees clean` falls through to unconditional `os.RemoveAll` with no prompt, `cp --force` truncates the source when src and dest resolve to the same file, `fresh`/`prune` force-remove dirty worktrees of merged branches, `project remove` always deletes `.git/modules` including unpushed history, the `sync` stale-ref fallback RemoveAlls non-empty submodule dirs and replaces them with non-submodule clones, and repair/workflow migrations clobber silently then auto-commit failures. Dungeon triage and the dungeon resolver violate the camp/fest ownership boundary by offering and adopting `festivals/` content. The excellent `internal/fsutil` atomic-write and lock primitives exist but roughly half the subsystems bypass them with bare `os.WriteFile` and unlocked read-modify-write cycles. Agent-facing contracts that festival-app depends on are broken or lying: `cr` flattens argv through `sh -c`, `doctor --json` exits 0 on failure, `flow items --json` never emits JSON, reads mutate state (workitem prune, intent migration, status cache), the stage filter hides design/explore items, and the praised scoped-commit engine commits worktree content rather than the index (`git commit --only` semantics). No CI exists and GitHub CI is intentionally paused for cost, so nothing currently stops regressions.

**Desired State:** Every finding in recommendations.md (R1-R36) and the second-pass additions (N-1 through N-27) is remediated on the `camp-hardening` branch in the linked worktree. The data-loss class is gone and container-tested. The gates are green and trustworthy in both profiles and enforced locally (just recipes plus a pre-push hook). The fest boundary is guarded by regression tests. All state writes are atomic and locked. The JSON and argv contracts festival-app shells are correct and versioned. The low-severity debt sweeps (fmt.Errorf ratchet, dead code, file splits, help polish) are done.

**Why This Matters:** camp is the daily driver for every campaign workspace and the shelled backend for festival-app. The destructive-command long tail sits directly under routine festival operation: one habitual `camp cp -f`, `camp fresh`, or `camp worktrees clean` can irrecoverably destroy uncommitted work, and the fest-boundary violations let camp triage fest-owned festival state into the dungeon. With the gate red on main and no CI, every merge compounds the problem. This festival is the launch-hardening pass that makes camp safe to operate and safe to build on.

## Scope

### In Scope

- All P0 items: R1-R10 plus the second-pass P0 additions N-1 (cp same-path truncation), N-2 (fresh/prune dirty-worktree removal), N-3 (project remove .git/modules deletion), N-6 (sync fallback RemoveAll and non-submodule reclone)
- All P1 hardening: R11-R18 (atomic-write sweep including N-21, registry and priority locking, dev-tagged test gate with local enforcement, dead tagged tests, container Reset, prune tests, reg alias, workitem list read purity) widened per second-pass section 4 (R18 covers N-8 and N-18; R6 requires N-4 and N-5 first; R10 covers N-19)
- All P2 contracts and consistency: R19-R28 plus N-7, N-9, N-11, N-13 through N-18, N-20, N-22 through N-26
- All P3 debt and polish: R29-R36 (R29 dead-code sweep strictly before R30 extraction)
- Local gate enforcement work: just recipes covering both build profiles, `go vet -tags=integration` in lint, and a pre-push hook
- Regression tests for every behavioral fix, with container integration tests for destructive paths
- Coordination output for the fest repo: a fest workitem capturing the commitkit scoped-commit mirror fix required by GIT-2/second-pass correction 2 (fest's `commit.go:541-553` reimplements the unscoped pattern independently of `SyncSubmoduleRef`)

### Out of Scope

- Any GitHub Actions workflow creation or modification: hosted CI is intentionally paused for cost; every gate enforcement mechanism in this festival is local (just recipes, pre-push hooks)
- Code changes inside the fest repo itself: camp owns `pkg/commitkit`, and the fest-side consumption fix is captured as a coordination workitem for fest, not implemented here
- New camp features or UX redesigns beyond what a finding requires (for example, the channel decision for `flow` in R9 picks between the two documented options, it does not design a new channel system)
- Re-litigating areas the second pass hunted and found clean (its "found clean" list: intent migration guards, quest rename rollback, links WithLock, jsoncontract determinism, etc.)

## Planned Phases

### 001_INGEST

Structure the copied review corpus (`input_specs/`: review-README.md, recommendations.md, second-pass-fable.md, seven findings docs) into `output_specs/` purpose, requirements, constraints, and context documents. The requirements doc must merge R1-R36 with the second-pass reordering: N-1/N-2/N-3/N-6 join P0, R6 is re-scoped behind N-4/N-5, R10 widens to moveFile/N-19, R18 widens to three read-mutation sites.

### 002_PLAN

Design the implementation phase and sequence breakdown honoring the dependency order from the review: data loss first, then trustworthy gates (R13-R15 unlock everything after), fest boundary, atomicity and locking, agent-facing contracts (festival-app coordination items grouped), then the cleanup sweeps with R29 before R30. Produce the sequence-level task design for the pending IMPLEMENT phase and the verification plan for POLISH.

### IMPLEMENT (pending, designed during 002_PLAN)

Execute the remediation sequences in the linked worktree `projects/worktrees/camp/camp-hardening` with per-sequence quality gates (testing, review, iterate, fest commit).

### POLISH (pending, designed during 002_PLAN)

Final verification sweep: run the full success-criteria checklist from FESTIVAL_GOAL.md, both-profile gates end to end, greps for residual `os.WriteFile` on state paths, and disposition of any remaining low findings in the decision log.

## Notes

- The festival is linked to a fresh worktree at `projects/worktrees/camp/camp-hardening` (branch `camp-hardening` from origin/main commit bc2ad1f). All code work happens there; this planning directory holds only festival documents.
- Authoritative work inventory: `001_INGEST/input_specs/recommendations.md` (R1-R36 with locations and approaches) and `001_INGEST/input_specs/second-pass-fable.md` (verification table, corrections, N-1 through N-27, and the required reordering in its section 4). The seven findings docs provide file:line detail per subsystem.
- The review baseline is commits 5c22d2b/11c270f; the worktree branches from bc2ad1f. Early ingest/plan work must re-confirm cited line numbers against bc2ad1f since intervening commits may have shifted them (the second pass already noted the two new commits did not touch finding sites, but bc2ad1f may be newer).
- camp test conventions: unit tests that mutate the filesystem use host `t.TempDir` per the repo's existing convention; the container harness under `tests/integration/` (testcontainers, `//go:build integration`) is for integration coverage of real git/filesystem effects. Destructive-path fixes need container tests.
- Both build profiles matter everywhere: stable omits dev-tagged code, `BUILD_TAGS=dev` includes it, and TEST-1 proved failures can hide in either.
