# Festival TODO - camp-hardening

**Goal**: Remediate all fable-review-2026-06-10 camp findings (R1-R36 plus N-1 through N-27): data-loss class first, then local gates for both build profiles, fest-boundary violations, the atomic-write and locking sweep, agent-facing contract bugs, and the low-severity cleanup sweeps.
**Status**: Planning

---

## Festival Progress Overview

### Phase Completion Status

- [x] 001_INGEST: structure the review corpus into purpose, requirements, constraints, and context specs (output_specs/ complete, workflow and gate approved)
- [x] 002_PLAN: design the implementation phase/sequence breakdown in dependency order (plan/STRUCTURE.md, plan/IMPLEMENTATION_PLAN.md, decisions D001-D007, IMPLEMENT and POLISH scaffolded)
- [ ] 003_IMPLEMENT: execute the 12 remediation sequences in the linked worktree (awaits Lance's review and promotion out of planning/)
- [ ] 004_POLISH: verification-only closeout per D006 (plan/VERIFICATION_CHECKLIST.md ready)

### Current Work Status

```
Active Phase: 002_PLAN (scaffold/validate steps)
Active Sequences: N/A (planning)
Blockers: None
```

---

## Phase Progress

### 001_INGEST

**Status**: Complete

- [x] output_specs/purpose.md, requirements.md (all R and N findings mapped with second-pass tiering), constraints.md, context.md
- [x] PRESENT checkpoint and phase gate approved (codex exec acted as user-proxy; one revision: R5/R6 retiered to P0)

### 002_PLAN

**Status**: In progress (workflow steps 1-6 complete and approved via user-proxy; scaffold done; validation pending)

- [x] inputs/gaps.md gap analysis
- [x] plan/STRUCTURE.md festival hierarchy with finding-to-sequence map
- [x] plan/IMPLEMENTATION_PLAN.md with per-sequence verification plans (both-profile gate definition)
- [x] decisions/D001-D007 recorded and indexed
- [x] 003_IMPLEMENT scaffolded: 12 sequences, 63 tasks
- [x] 004_POLISH scaffolded with plan/VERIFICATION_CHECKLIST.md

### 003_IMPLEMENT

**Status**: Not started (planned)

#### Sequences

- [ ] 01_gate_restoration (R1; D007 bootstrap)
- [ ] 02_destructive_command_safety (R2, N-1, N-2, R16, N-3)
- [ ] 03_sync_and_migration_safety (N-6, N-10, R7, R4, R10, N-19, N-12)
- [ ] 04_gate_enforcement (R13, R14, R15; D005)
- [ ] 05_fest_boundary (R3, WF-7)
- [ ] 06_atomicity_and_locking (R11, N-21, R12)
- [ ] 07_commit_semantics (N-4, N-5, R6, N-14, N-23; D001)
- [ ] 08_argv_and_read_paths (R5, R18, N-8, N-18)
- [ ] 09_app_contracts (R8, N-7, N-9, R20, R26, N-15, N-16, N-11, R21, N-25, N-17, N-24; D004)
- [ ] 10_cli_consistency (R19, R22, R23, R17, R9, R27, R28, N-13; D002)
- [ ] 11_debt_burn_down (R24, R29 before R30, R31, R25, R32, R33, N-20, N-22, N-26, N-27; D003)
- [ ] 12_test_hygiene_and_docs (R34, R35, R36, ARCH-9, BR-8)

### 004_POLISH

**Status**: Not started (verification checklist prepared)

- [ ] Execute plan/VERIFICATION_CHECKLIST.md with recorded evidence
- [ ] DISPOSITION_LOG.md and COMPLETION_REPORT.md

---

## Blockers

None currently.

---

## Decision Log

- D001: N-4 `--only` fix via temporary GIT_INDEX_FILE snapshot; staged-blob semantics for `--staged` (2026-06-12)
- D002: flow stays dev-channel; scaffolding and docs become channel-aware via the Profile constant (2026-06-12)
- D003: surgical P0 migration fixes first; shared statusdir primitive lands in debt burn-down (2026-06-12)
- D004: schema-bump list (intents/flow-items/status-all/concepts/version new; workitems v1alpha6; manifest bump) plus docs rows plus festival-app coordination workitem (2026-06-12)
- D005: pre-push hook execs just gate-fast with CAMP_GATE_FULL upgrade; just gate is the full both-profile matrix; no GitHub Actions (2026-06-12)
- D006: POLISH is verification-only; all code sweeps live in IMPLEMENT (2026-06-12)
- D007: R1 gate restoration runs as a scope-capped bootstrap before the data-loss sequences (2026-06-12)
- Gate decisions during planning were made via the documented user-proxy pattern (codex exec); recorded in workflow history: INGEST PRESENT (one rejection, revised, approved), INGEST phase gate (3 approvals), PLAN PRESENT (one rejection, revised, approved)

---

*Detailed progress available via `fest status`*
