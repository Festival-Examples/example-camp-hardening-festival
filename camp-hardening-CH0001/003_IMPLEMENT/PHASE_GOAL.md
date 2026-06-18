---
fest_type: phase
fest_id: 003_IMPLEMENT
fest_name: IMPLEMENT
fest_parent: camp-hardening-CH0001
fest_order: 3
fest_status: completed
fest_created: 2026-06-11T23:24:20.757211-06:00
fest_updated: 2026-06-15T16:59:38.688646-06:00
fest_phase_type: implementation
fest_tracking: true
---


# Phase Goal: 003_IMPLEMENT

**Phase:** 003_IMPLEMENT | **Status:** Pending | **Type:** Implementation

## Phase Objective

**Primary Goal:** Execute all remediation sequences for R1-R36 and N-1..N-27 in dependency order (gate restoration bootstrap, data loss, gate enforcement, fest boundary, atomicity and locking, commit semantics, argv and read paths, app contracts, CLI consistency, debt burn-down, test hygiene and docs) in the linked worktree, with both-profile quality gates per sequence

**Context:** This phase executes the plan produced in 002_PLAN (plan/IMPLEMENTATION_PLAN.md, plan/STRUCTURE.md, decisions D001-D007) against the linked worktree `projects/worktrees/camp/camp-hardening` (branch point bc2ad1f). The sequence order is load-bearing: 01 restores a green test baseline (D007 bootstrap exception), 02-03 eliminate the data-loss class, 04 installs trustworthy both-profile local gates (D005; no GitHub Actions), and everything after runs under those gates. 004_POLISH consumes this phase's output for verification-only closeout (D006).

## Required Outcomes

Deliverables this phase must produce:

- [ ] Every R1-R36 and N-1..N-27 finding fixed in the worktree or explicitly dispositioned in the decision log (disposition not available for the data-loss, gate, boundary, or festival-app-contract classes)
- [ ] Regression tests for every behavioral fix; container integration tests for every destructive-path fix (worktrees clean, cp, fresh/prune, project remove, sync fallback)
- [ ] Local gate enforcement live: `just gate` / `just gate-fast` recipes, BUILD_TAGS-aware tests and lint, `go vet -tags=integration ./...` in lint, committed pre-push hook with `just hooks install`
- [ ] Versioned JSON contract changes per D004 with docs/json-contracts.md rows and the festival-app coordination workitem
- [ ] The fest commitkit-mirror coordination workitem created (second-pass correction 2; C-5)
- [ ] Zero `os.WriteFile` on the R11/N-21 state paths; registry and priority-store mutations locked

## Quality Standards

Quality criteria for all work in this phase:

- [ ] Every sequence closes with build, test, and lint green in BOTH profiles (stable and `BUILD_TAGS=dev`), run explicitly, not inferred
- [ ] No `.github/workflows/` files created or modified anywhere in the phase
- [ ] Verified-negatives from the review (constraints C-3) are not re-fixed or re-litigated
- [ ] All commits use `fest commit`; behavior changes and extraction-only changes land in separate commits
- [ ] File:line targets re-verified against the worktree before editing (review citations are from 5c22d2b/11c270f; the worktree is at bc2ad1f)

## Sequence Alignment

| Sequence | Goal | Key Deliverable |
|----------|------|-----------------|
| 01_gate_restoration | Green unit gate in both profiles (R1; D007 scope cap) | Manifest allowlist fix plus recorded baseline survey |
| 02_destructive_command_safety | Commands that delete files cannot destroy uncommitted work (R2, N-1, N-2, R16, N-3) | Guarded worktrees clean/cp/prune/remove with container tests |
| 03_sync_and_migration_safety | Sync/pull and migrations all-or-nothing, never clobber (N-6, N-10, R7, R4, R10, N-19, N-12) | Quarantine fallback, pre-validated migrations, guarded moveFile |
| 04_gate_enforcement | Trustworthy local both-profile gates (R13, R14, R15; D005) | just gate/gate-fast, pre-push hook, integration vet in lint, container Reset fix |
| 05_fest_boundary | camp never touches fest-owned state (R3, WF-7) | Dungeon exclusions and resolver guard with regression tests |
| 06_atomicity_and_locking | Atomic state writes, locked read-modify-write (R11, N-21, R12) | fsutil sweep, UpdateRegistry, priority.WithLock, TOCTOU fix |
| 07_commit_semantics | Scoped commits commit exactly what they claim (N-4, N-5, R6, N-14, N-23; D001) | Temp-index engine, rename-safe staging, scoped sync commits, fest workitem |
| 08_argv_and_read_paths | Argv preserved end to end; reads never write (R5, R18, N-8, N-18) | Direct just exec, positional flow args, prune/migration/cache moved to write paths |
| 09_app_contracts | Every app-shelled surface versioned and correct (R8, N-7, N-9, R20, R26, N-15, N-16, N-11, R21, N-25, N-17, N-24; D004) | Versioned JSON contracts, manifest completeness, profile visibility |
| 10_cli_consistency | One flag idiom, exit-code table, signal context, honest docs (R19, R22, R23, R17, R9, R27, R28, N-13; D002) | Standardized flags, channel-aware scaffolding, shell/nav fixes |
| 11_debt_burn_down | Structural debt removed in safe order (R24, R29 before R30, R31, R25, R32, R33, N-20, N-22, N-26, N-27; D003) | Ratchet lint, dead-code deletion, extractions, shared move primitive |
| 12_test_hygiene_and_docs | Suite reliability and docs polish (R34, R35, R36, ARCH-9, BR-8) | Deflaked tests, regenerated reference, misc fixes, upstream tags |

## Pre-Phase Checklist

Before starting implementation:

- [ ] Planning phase complete
- [ ] Architecture/design decisions documented
- [ ] Dependencies resolved
- [ ] Development environment ready

## Phase Progress

### Sequence Completion

- [ ] 01_gate_restoration
- [ ] 02_destructive_command_safety
- [ ] 03_sync_and_migration_safety
- [ ] 04_gate_enforcement
- [ ] 05_fest_boundary
- [ ] 06_atomicity_and_locking
- [ ] 07_commit_semantics
- [ ] 08_argv_and_read_paths
- [ ] 09_app_contracts
- [ ] 10_cli_consistency
- [ ] 11_debt_burn_down
- [ ] 12_test_hygiene_and_docs

## Notes

Hard constraints carried from FESTIVAL_RULES.md and 001_INGEST/output_specs/constraints.md: local gates only (no GitHub Actions, never reference re-enabling it); both profiles at every gate; verified-negatives are do-not-fix; commitkit is fest's public API and the fest-side mirror is a workitem, not camp scope; `fest commit` for every commit; new unit tests follow the host t.TempDir camp convention, destructive-path tests go in tests/integration (container harness); the `lint-no-host-fs-tests` ratchet must not grow.

---

*Implementation phases use numbered sequences. Create sequences with `fest create sequence`.*