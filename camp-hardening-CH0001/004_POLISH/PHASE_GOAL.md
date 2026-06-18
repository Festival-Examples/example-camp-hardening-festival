---
fest_type: phase
fest_id: 004_POLISH
fest_name: POLISH
fest_parent: camp-hardening-CH0001
fest_order: 4
fest_status: completed
fest_created: 2026-06-11T23:24:20.777497-06:00
fest_phase_type: planning
fest_tracking: true
---

# Phase Goal: 004_POLISH

**Phase:** 004_POLISH | **Status:** Completed | **Type:** Planning

## Phase Objective

**Primary Goal:** Verification-only closeout per decision D006: run the final verification checklist mapping every FESTIVAL_GOAL success criterion to recorded command output (both-profile gates, integration vet, os.WriteFile grep, no-GitHub-Actions check, disposition review, regression-test audit)

**Context:** All code work lives in 003_IMPLEMENT (D006); this phase contains NO code changes. It exists so festival completion is an auditable artifact: every FESTIVAL_GOAL functional and quality criterion gets checked against the final worktree state with the actual command output recorded, and every P2/P3 finding is confirmed fixed (commit link) or dispositioned with rationale. If verification finds a miss, the fix returns to the owning IMPLEMENT sequence (reopened); it is never patched here.

## Exploration Topics

What areas need to be explored during this phase:

- The final festival diff (`git diff bc2ad1f..HEAD --stat` in the worktree) for scope review: no `.github/workflows/` entries, no unexplained files
- Residual grep surface for the R11/N-21 atomic-write sweep and any new `os.WriteFile` introduced after sequence 06
- The decision log and per-sequence task notes for every dispositioned P2/P3 item

## Key Questions to Answer

Questions that must be answered before this phase is complete:

- Does `just gate` (build, test, lint in BOTH profiles, per D005) pass end to end on the final tree, with output recorded?
- Is `go vet -tags=integration ./...` clean and wired into `just lint`?
- Is every FESTIVAL_GOAL "Functional Success", "Quality Success", and "Complete When" checkbox satisfiable with evidence, and checked off?
- Does every destructive-path fix (worktrees clean, cp, fresh/prune, project remove, sync fallback) have a container integration test, and every data-loss/boundary/contract behavioral fix a regression test?
- Is every P2/P3 finding fixed (commit reference) or dispositioned with rationale, with zero dispositions in the data-loss, gate, boundary, and app-contract classes?
- Are docs/json-contracts.md rows and the JSON-6 stability matrix current, and do the festival-app and fest coordination workitems exist?

## Expected Documents

Documents that will be produced during this phase:

- `VERIFICATION_CHECKLIST.md`: the executable checklist (created during planning, executed here) with a recorded-output column per item
- `DISPOSITION_LOG.md`: the P2/P3 fixed-or-dispositioned table with commit references and rationale
- `COMPLETION_REPORT.md`: summary for Lance: what shipped, what was dispositioned and why, gate evidence, coordination workitems opened

## Success Criteria

This planning phase is complete when:

- [x] VERIFICATION_CHECKLIST.md fully executed with command output recorded for every item, all items passing
- [x] `just gate` green end to end in both profiles on the final worktree tree
- [x] `go vet -tags=integration ./...` clean
- [x] Zero `os.WriteFile` on the R11/N-21 state paths (grep output recorded)
- [x] `git diff bc2ad1f..HEAD --stat` shows no `.github/workflows/` changes
- [x] DISPOSITION_LOG.md covers every R19-R36 and remaining-N item; data-loss/gate/boundary/contract classes are fix-only
- [x] Regression and container test presence audit recorded per destructive-path and behavioral fix
- [x] COMPLETION_REPORT.md written and every FESTIVAL_GOAL checkbox updated

## Notes

Checklist source of truth: FESTIVAL_GOAL.md success criteria plus 002_PLAN/plan/IMPLEMENTATION_PLAN.md's POLISH section and D006's eight-point list. Constraints C-1 (no GitHub Actions), C-2 (both profiles), C-3 (verified-negatives untouched) all re-verified here as checklist items. This phase does not promote or release anything; release/tagging remains a separate human decision after Lance reviews the completion report.

---

*Planning phases use freeform structure. Create topic directories as needed.*
