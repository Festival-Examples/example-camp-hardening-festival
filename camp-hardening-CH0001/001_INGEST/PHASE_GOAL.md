---
fest_type: phase
fest_id: 001_INGEST
fest_name: INGEST
fest_parent: camp-hardening-CH0001
fest_order: 1
fest_status: completed
fest_created: 2026-06-11T22:21:20.371564-06:00
fest_updated: 2026-06-11T23:09:37.33071-06:00
fest_phase_type: ingest
fest_tracking: true
---


# Phase Goal: 001_INGEST

**Phase:** 001_INGEST | **Status:** Pending | **Type:** Ingest

## Phase Objective

**Primary Goal:** Ingest and structure input materials into actionable specifications

**Context:** The input is the complete staff-level review of the camp repo at `workflow/explore/fable-review-2026-06-10/camp/`, copied verbatim into `input_specs/`: the review index (review-README.md), the prioritized remediation list recommendations.md (R1-R36 with severity, location, approach, and step counts), the adversarial second pass second-pass-fable.md (20/20 findings confirmed, three corrections, new findings N-1 through N-27, and a required P0 reordering in its section 4), and seven per-area findings docs with file:line evidence. The structured output_specs/ feed the 002_PLAN phase, which designs the implementation phase and sequence breakdown for remediating every finding in the linked worktree `projects/worktrees/camp/camp-hardening`.

## Input Sources

Place all raw input materials in `input_specs/`:

- [ ] `recommendations.md`: the authoritative prioritized work inventory, R1-R36 in four priority tiers with suggested festival phasing
- [ ] `second-pass-fable.md`: adversarial verification of all critical/high findings, corrections to R2/R6/R10, new findings N-1 through N-27 (including cp --force truncation, fresh/prune dirty-worktree removal, project remove git-modules deletion, and git commit --only semantics), and the required reordering of the remediation plan in its section 4
- [ ] `review-README.md`: review scope, methodology, severity summary (3 critical, ~24 high, ~45 medium, ~60 low), what is in good shape, and the findings-doc index
- [ ] `findings-architecture.md`, `findings-command-ux.md`, `findings-subsystems.md`, `findings-error-handling-standards.md`, `findings-testing.md`, `findings-build-release.md`, `findings-json-contracts.md`: per-area evidence with file:line citations backing every R and N item

## Expected Outputs

The following structured documents will be created in `output_specs/`:

| Output | Purpose |
|--------|---------|
| `purpose.md` | Festival purpose, success criteria, motivation |
| `requirements.md` | Prioritized requirements (P0/P1/P2) with traceability |
| `constraints.md` | Technical and process constraints |
| `context.md` | Prior art, related systems, key references |

## Success Criteria

This ingest phase is complete when:

- [ ] All input sources reviewed and understood
- [ ] Output specs created following standard structure
- [ ] User has approved the structured output
- [ ] No unresolved questions or ambiguities

## Workflow

This phase uses step-based workflow guidance. See `WORKFLOW.md` for the step-by-step process.

Use `fest next` to see the current step.
Use `fest workflow advance` to move to the next step.

## Notes

- requirements.md must merge R1-R36 with the second-pass section 4 reordering: N-1, N-2, N-3, and N-6 join P0 alongside R2; R6 is re-scoped to land N-4 (--only worktree semantics) and N-5 (staged-rename corruption) first or simultaneously; R10 widens to guard `moveFile` itself plus EXDEV gating (N-19); R18 widens to three read-mutation sites (workitem prune, intent EnsureDirectories migration N-8, status-all cache write N-18); R11's sweep list gains `internal/config/allowlist.go:84` (N-21).
- constraints.md must record the festival rules: fest commit only, work in the linked worktree, both build profiles (stable and BUILD_TAGS=dev) must pass gates, local enforcement only (just recipes and pre-push hooks, no GitHub Actions), unit tests use host t.TempDir per repo convention with destructive paths container-tested.
- The review cites line numbers at commits 5c22d2b/11c270f; the worktree branch point is bc2ad1f. Treat citations as approximate until re-confirmed in the worktree.
- The second pass's "hunted and found clean" list is in scope for context.md so planning does not re-litigate those areas.

---

*Ingest phases transform unstructured input into structured specifications ready for planning.*