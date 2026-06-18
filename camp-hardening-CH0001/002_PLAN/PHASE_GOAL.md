---
fest_type: phase
fest_id: 002_PLAN
fest_name: PLAN
fest_parent: camp-hardening-CH0001
fest_order: 2
fest_status: completed
fest_created: 2026-06-11T22:21:20.374287-06:00
fest_updated: 2026-06-12T00:59:59.304137-06:00
fest_phase_type: planning
fest_tracking: true
---


# Phase Goal: 002_PLAN

**Phase:** 002_PLAN | **Status:** Completed | **Type:** Planning

## Phase Objective

**Primary Goal:** Plan architecture, design decisions, and task breakdown

**Context:** The remediation inventory is large (R1-R36 plus N-1 through N-27, roughly 130 findings across four priority tiers) and heavily interdependent: the gate work (R13-R15) must land early so every later fix is verifiable in both build profiles; N-4 and N-5 must precede R6 because the scoped-commit fix builds on `--only` semantics that are currently wrong; R29 dead-code deletion must precede R30 extraction; several contract fixes coordinate with festival-app. This phase converts the structured specs from 001_INGEST into a concrete implementation phase and sequence breakdown (the pending IMPLEMENT and POLISH phases in fest.yaml) with explicit dependency ordering, per-sequence scope, and per-sequence verification plans, so implementation can run autonomously in the linked worktree `projects/worktrees/camp/camp-hardening`.

## Exploration Topics

What areas need to be explored during this phase:

- Re-confirm every cited file:line against the worktree branch point bc2ad1f (review citations are from 5c22d2b/11c270f) and note any drift in the plan documents
- Sequence design for the data-loss class: which fixes share fixtures (R7 and N-6 both need a hostile-remote container fixture; N-2 folds into R16's prune test work; N-1 needs same-file and dir-into-itself container cases)
- Gate design: what `just test dev`, both-profile release gating, `go vet -tags=integration` in lint, and the pre-push hook look like concretely, given GitHub Actions is off the table
- The scoped-commit cluster ordering: N-4 index-snapshot approach vs divergence-check approach, N-5 rename parsing, then R6 scoping across syncParentRef/SyncSubmoduleRef/refs-sync and N-14 root-commit scoping, plus the fest coordination workitem for fest's own commit.go reuse
- Atomic-write and locking sweep inventory: enumerate every os.WriteFile site on state paths (R11 list plus N-21) and every unlocked read-modify-write (R12) as a checklist the implementation tasks consume
- festival-app contract grouping: which of N-7, N-9, N-11, N-15, N-16, N-17, R19, R20, R21, R26 need schema bumps or app-side coordination, and in what order
- The R9 flow-channel decision: promote `flow` to stable vs channel-aware scaffolded skills
- R25 shared statusdir move primitive design (collision policy, dated buckets, cross-device fallback) and whether it lands before or after the individual P0 migration fixes

## Key Questions to Answer

Questions that must be answered before this phase is complete:

- How many implementation sequences, in what order, and which findings map to each, such that every sequence leaves both-profile gates green (no sequence may end with a red gate)?
- Does N-4 get fixed via a temporary GIT_INDEX_FILE snapshot or via a `git diff --quiet` divergence guard, and what does that choice mean for every auto-commit call site?
- What is the exact pre-push hook content and which just recipes does it call, so it stays fast enough to run on every push?
- Which JSON contract changes bump schema versions, and how is festival-app notified (workitem, docs/json-contracts.md rows, or both)?
- For R9, does `flow` promote to stable or do scaffolded skills become channel-aware?
- What goes in the POLISH phase versus the tail of IMPLEMENT (the R29-R36 sweeps need explicit homes)?

<!-- Add more questions as they emerge -->

## Expected Documents

Documents that will be produced during this phase:

- `plan/`: the implementation plan, finding-to-sequence mapping table covering all of R1-R36 and N-1 through N-27, dependency graph, and per-sequence verification plans
- `decisions/`: decision records for the open calls (N-4 approach, R9 flow channel, R25 primitive timing, schema-bump list), indexed in decisions/INDEX.md
- Scaffolded IMPLEMENT phase (phases, sequences, and tutorial-grade task documents created via fest create) plus the POLISH phase verification checklist

<!-- Add more documents as planning progresses -->

## Success Criteria

This planning phase is complete when:

- [x] Every finding in requirements.md (all of R1-R36 and N-1 through N-27) is mapped to exactly one implementation sequence or an explicit dispositioned decision record; nothing is deferred out of the festival
- [x] The IMPLEMENT phase is scaffolded with sequences in dependency order (data loss, gates, fest boundary, atomicity/locking, contracts, cleanup with R29 before R30) and tutorial-grade task documents
- [x] Each sequence's verification plan names the concrete commands (both-profile just build/test/lint, targeted go test runs, container integration runs) and the regression tests it must add
- [x] The open decisions (N-4 approach, R9 flow channel, R25 timing, schema bumps) are recorded in decisions/ with rationale
- [x] The POLISH phase has a final verification checklist matching FESTIVAL_GOAL.md success criteria, including the residual os.WriteFile grep and the no-GitHub-Actions check

<!-- Add more success criteria as they become clear -->

## Notes

- Inputs: `001_INGEST/output_specs/` (purpose, requirements, constraints, context) and the raw review docs in `001_INGEST/input_specs/` for file:line detail.
- Constraints carried from FESTIVAL_RULES.md: fest commit only, linked-worktree execution, both build profiles gate every sequence, local enforcement only (no GitHub Actions), host t.TempDir unit-test convention with container tests for destructive paths.
- The fest-repo mirror of the commitkit scoping fix is a coordination workitem, not an implementation sequence here; plan the workitem content during this phase.
- Quality gates (testing, review, iterate, fest-commit) auto-append to implementation sequences per fest.yaml; sequence task design should not duplicate them.

---

*Planning phases use freeform structure. Create topic directories as needed.*