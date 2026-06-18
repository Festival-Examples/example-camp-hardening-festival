# Disposition Log: camp-hardening (CH0001)

Finalized 2026-06-15 during `004_POLISH`.

## Policy

- P0/P1 findings are fix-only.
- Data-loss, gate, boundary, and app-contract findings are fix-only.
- P2/P3 findings may be fixed or explicitly dispositioned with rationale.

## Summary

| Class | Findings | Disposition |
|---|---|---|
| P0 data-loss / boundary / red-gate | R1-R10, R16, N-1, N-2, N-3, N-6, N-10, N-12, N-19 | Fixed in implementation sequences 01-05 and 07; regression coverage verified by final `just gate` and `just test integration`. |
| P1 durability / gates / reads | R11-R18, N-4, N-5, N-8, N-14, N-18, N-21, N-23 | Fixed in implementation sequences 04, 06, 07, and 08; final gate passed. |
| P2 app contracts / CLI consistency | R19-R28, N-7, N-9, N-11, N-13, N-15, N-16, N-17, N-24, N-25 | Fixed in implementation sequences 09 and 10; festival-app coordination workitem `WI-3db3f8` exists. |
| P3 debt and polish | R29-R36, N-20, N-22, N-26, N-27, ARCH-9 | Fixed in implementation sequences 11 and 12; upstream tags pinned at `guild-scaffold v0.1.0` and `obey-shared v0.4.4`. |

## Explicit Closeout Notes

### `os.WriteFile` Remainder

The final grep still reports `os.WriteFile` in a small expected-remainder set. This is not a skipped R11/N-21 state-path fix; sequence 06 task 02 recorded these as legitimate non-state, migration, feedback, or output-file writes:

- `internal/intent/promote/promote.go`: promote README scaffold write.
- `internal/intent/feedback/tracker.go`: feedback tracker.
- `internal/intent/feedback/builder.go`: feedback intent builder.
- `internal/intent/migration_filesystem.go` and `internal/intent/migration.go`: migration utilities.
- `internal/scaffold/init.go`: initial `.gitignore` creation.
- `internal/commands/workitem/workitem.go`: explicit path-output file.
- `cmd/camp/gendocs.go`: documentation generation output.

Zero hits remain in the required converted R11/N-21 state files listed in `VERIFICATION_CHECKLIST.md`.

### Inherited Non-FE Commits

The range `bc2ad1f..HEAD` contains five inherited shell commits without the full `FE-CH0001` task suffix. They predate the active CH0001 implementation sequence and were not rewritten. All CH0001 task commits generated during the festival carry `[OBEY-CAMPAIGN-8deed8b4-FE-CH0001]`.

## Coordination Workitems

- `WI-64af81`: `design-fest-ingest-capability-2026-06-15`.
- `WI-25121c`: `design-fest-commitcampaignroot-scope-unscoped-commit-2026-06-15`.
- `WI-3db3f8`: `design-festival-app-camp-contract-changes-2026-06-15`.

## Final Disposition

No high-risk finding is left intentionally unfixed in camp. Remaining work is cross-project adoption or release decision work captured by coordination workitems.
