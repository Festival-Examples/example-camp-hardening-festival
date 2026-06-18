# D006: POLISH Phase Scope vs IMPLEMENT Tail

**Status:** accepted
**Date:** 2026-06-12

## Context

fest.yaml defines pending phases IMPLEMENT (implementation) and POLISH (planning). The R29-R36 sweeps could live at the tail of IMPLEMENT or inside POLISH. FESTIVAL_GOAL describes POLISH as "verify the full success-criteria checklist, close the low-severity sweeps, and confirm both-profile gates end to end".

## Options

### Option A: Code sweeps in POLISH
- **Pros:** Smaller IMPLEMENT phase.
- **Cons:** POLISH is a planning-type phase with no auto-appended quality gates (testing/review/iterate/fest-commit); code changes there would dodge exactly the gate discipline the festival exists to install. "Close the low-severity sweeps" reads as verify-closed, not implement-there.

### Option B: All code changes in IMPLEMENT; POLISH is verification-only
- **Pros:** Every code change passes through implementation quality gates; POLISH becomes a clean, auditable closeout (checklist runs, grep gates, disposition review) matching its planning type.
- **Cons:** IMPLEMENT is large (12 sequences).

## Decision

**Option B.** All R29-R36 and remaining-N code work lives in IMPLEMENT sequences 11 and 12. POLISH contains no code changes; it executes the final verification checklist:

1. `just gate` (both-profile build, test, lint) end to end on the final tree.
2. `go vet -tags=integration ./...` clean.
3. Residual grep: zero `os.WriteFile` on the R11/N-21 state paths.
4. No `.github/workflows/` additions or modifications anywhere in the festival diff.
5. Every FESTIVAL_GOAL functional and quality criterion checked off with the command output recorded.
6. Disposition review: every P2/P3 finding either fixed (link to commit) or dispositioned with rationale in the decision log; data-loss, gate, boundary, and app-contract classes confirmed fix-only.
7. Regression-test presence audit for every destructive-path fix (container tests) and behavioral fix.
8. docs/json-contracts.md stability matrix (JSON-6) current.

## Consequences

- 003_IMPLEMENT carries all 12 sequences; 004_POLISH is scaffolded as a planning phase containing the verification checklist document consumed at execution time.
- If POLISH verification finds a miss, the fix returns to an IMPLEMENT sequence (reopened), keeping gates on code.
