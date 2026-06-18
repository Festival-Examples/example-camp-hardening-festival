# D007: Gate-Restoration Bootstrap Exception (R1 Before the Data-Loss Sequences)

**Status:** accepted
**Date:** 2026-06-12

## Context

FESTIVAL_GOAL orders the work "data-loss class first, then gate enforcement". The plan nevertheless places sequence 01_gate_restoration (R1 only: the manifest allowlist fix) before the data-loss sequences 02 and 03. This needs to be an explicit, justified exception rather than silent reordering.

## Options

### Option A: Strict goal order (data loss first, R1 later with R13-R15)
- **Pros:** Literal fidelity to the goal sentence.
- **Cons:** The unit gate is RED on main (TEST-1). Every implementation sequence ends with an auto-appended testing quality gate that must pass; with the suite red, sequence 02's gate either fails on the pre-existing manifest failure or gets waved through over a red baseline, normalizing exactly the gate-rot the festival exists to fix. Data-loss regression tests added in 02/03 would land into a suite that cannot run green.

### Option B: R1 as a bootstrap exception, full gate enforcement (R13-R15) staying after data loss
- **Pros:** Two-task, near-zero-risk sequence (one allowlist entry plus count bump, plus a no-fix baseline survey) makes the suite green so the data-loss sequences' own quality gates are meaningful from the first fix onward. The substantive gate-enforcement work (R13-R15) remains AFTER the data-loss sequences, preserving the goal's ordering for everything with real scope.
- **Cons:** A one-sequence deviation from the literal goal sentence.

## Decision

**Option B.** R1 is a bootstrap precondition for trustworthy verification, not "gate enforcement" in the goal's sense; the goal's gate-enforcement clause (R13-R15, recipes, hook) stays in sequence 04 after the data-loss sequences 02 and 03. The data-loss class therefore remains the first substantive work, and no data-loss fix ever lands against a red baseline.

## Consequences

- Sequence 01 is scope-capped: the manifest allowlist fix and a record-only baseline survey. Any other failure discovered in the baseline is logged for its owning sequence, not fixed in 01.
- STRUCTURE.md and IMPLEMENTATION_PLAN.md reference this decision in their dependency tables.
