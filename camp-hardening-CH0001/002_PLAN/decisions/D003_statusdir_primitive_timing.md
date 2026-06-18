# D003: R25 Shared Statusdir Move Primitive Timing

**Status:** accepted
**Date:** 2026-06-12

## Context

The repo contains four parallel implementations of "move a markdown item between status directories" with divergent collision policies (workflow service_move, dungeon service_move, intent helpers moveFile, plus the two no-check movers in repair and sync-migrate). R25 proposes one shared primitive (no-replace semantics, dated buckets, cross-device fallback). Meanwhile the P0 sequence 03 must fix the specific data-loss movers (WF-3, WF-4, R10/N-19) immediately. Question: build the primitive first and have P0 consume it, or fix P0 surgically and consolidate later?

## Options

### Option A: Primitive first
- **Pros:** P0 fixes land on the final abstraction; no later migration.
- **Cons:** Blocks data-loss elimination behind a P3-sized design (collision policy, dated buckets, EXDEV fallback, four call-site migrations); a new shared abstraction is the riskiest thing to write while the gates are still untrustworthy (sequence 04 has not landed yet).

### Option B: Surgical P0 fixes first, primitive later in debt burn-down
- **Pros:** Data loss stops in sequence 03 with minimal, reviewable diffs reusing existing checks (`statuspath.ExistingItemPath`, `service_move.go`'s destination check, O_EXCL); the primitive (11.05) then consolidates with trustworthy both-profile gates and the P0 regression tests as its safety net.
- **Cons:** The movers are touched twice; some duplication persists between sequences 03 and 11.

## Decision

**Option B.** Sequence 03 fixes WF-3/WF-4/R10/N-19 surgically with the narrowest correct change; task docs for 03 require the fixes be written as small helper functions with no-replace semantics so 11.05's migration into the shared primitive is mechanical. 11.05 builds the primitive, migrates all four implementations (plus the dungeon TOCTOU and stat-conflation fixes), and deletes the interim helpers.

Rationale: data loss is the festival's first priority; the P0 regression tests written in sequence 03 become exactly the conformance suite the primitive must pass in 11.05, making the consolidation safe rather than speculative.

## Consequences

- Sequence 03 tasks note the "structure for later extraction" requirement explicitly.
- 11.05's task consumes the sequence-03 regression tests as its acceptance baseline and additionally decides whether schema-driven workflow supersedes hardcoded dungeon (recorded in its task output, not pre-decided here).
