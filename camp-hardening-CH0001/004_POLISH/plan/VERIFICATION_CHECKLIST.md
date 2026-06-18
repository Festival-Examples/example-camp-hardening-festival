# Final Verification Checklist: camp-hardening (CH0001)

Executed on 2026-06-15 against linked worktree `projects/worktrees/camp/camp-hardening`.

- Branch: `camp-hardening`
- Pushed PR: https://github.com/Obedience-Corp/camp/pull/324
- Final pushed head: `5db31ec9753d1c44a3e6dd69229223405a16cfc4`
- Verification policy: code work stayed in `003_IMPLEMENT`; `004_POLISH` only records evidence.

## A. Gates

| # | Check | Evidence |
|---|---|---|
| A1 | Stable build green | `GOFLAGS=-buildvcs=false just gate` ran stable build: 99 packages, build successful. |
| A2 | Dev build green | `GOFLAGS=-buildvcs=false just gate` ran `BUILD_TAGS=dev` build: 101 packages, build successful. |
| A3 | Stable unit suite green | `GOFLAGS=-buildvcs=false just gate`: stable unit passed, 81 packages, 2544/2544 tests. |
| A4 | Dev unit suite green | `GOFLAGS=-buildvcs=false just gate`: dev unit passed, 83 packages, 2565/2565 tests. |
| A5 | Stable lint green | `GOFLAGS=-buildvcs=false just gate`: stable lint reported `0 issues`; host-fs and fmt.Errorf ratchets clean. |
| A6 | Dev lint green | `GOFLAGS=-buildvcs=false just gate`: dev lint reported `0 issues`; host-fs and fmt.Errorf ratchets clean. |
| A7 | Integration vet clean and wired into lint | `justfile` lint recipe includes `go vet -tags=integration ./...`; `just gate` ran `vet integration` successfully. |
| A8 | Container integration suite green | `GOFLAGS=-buildvcs=false just test integration`: 1 suite, 373/373 tests passed in 164.35s. |
| A9 | Full gate recipe green end to end | `GOFLAGS=-buildvcs=false just gate`: completed with `=== gate: PASSED ===`. |
| A10 | Pre-push hook present and installable | `.githooks/pre-push` delegates to `just gate-fast` or `CAMP_GATE_FULL=1 just gate`; `just hooks-install` set `core.hooksPath` to `.githooks`. |
| A11 | No GitHub Actions changes | `git diff bc2ad1f..HEAD --stat -- .github` produced no output; `git ls-files .github .github/workflows` produced no output. |

## B. Data-Loss Class

| # | Check | Evidence |
|---|---|---|
| B1 | `worktrees clean` destructive guards | Sequence 02 tasks complete; integration suite includes worktree-clean cases and final 373/373 integration pass. |
| B2 | `camp cp --force` same-file and self-recursion guards | Sequence 02 task 02 complete; container integration suite passed. |
| B3 | `fresh`/`prune` dirty-worktree protection and prune coverage | Sequence 02 tasks 03-04 complete; unit plus integration suite passed. |
| B4 | `project remove` unpushed-ref guard | Sequence 02 task 05 complete; container integration suite passed. |
| B5 | `sync` stale-ref fallback safety and drift correction | Sequence 03 tasks 01-02 complete; sync integration coverage passed in final suite. |
| B6 | Repair/workflow migrations all-or-nothing | Sequence 03 tasks 04-05 complete; sequence gates and final gate passed. |
| B7 | Intent move guard and promote rollback | Sequence 03 task 06 complete; sequence gates and final integration passed. |
| B8 | `pull all` does not abort pre-existing rebase | Sequence 03 task 03 complete; final integration suite passed. |

## C. Boundary, Atomicity, Locking

| # | Check | Evidence |
|---|---|---|
| C1 | Dungeon excludes `festivals`/`projects`; resolver skips fest-owned dungeon | Sequence 05 task 01 complete; final unit/integration gates passed. |
| C2 | Promote ingest boundary and coordination workitem | Sequence 05 task 02 complete; `camp workitem list --json` shows `design-fest-ingest-capability-2026-06-15` (`WI-64af81`). |
| C3 | `os.WriteFile` state-path sweep | Final grep found only the expected remainder documented in sequence 06 task 02: promote README scaffold, feedback tracker/builder, intent migration utilities, initial `.gitignore` creation, workitem path-output file, and gendocs. Zero hits remain in required converted files: `internal/intent/service.go`, `internal/quest/quest.go`, `internal/workflow/service_init.go`, `internal/workflow/service_sync_migrate.go`, `internal/flow/registry.go`, `internal/config/allowlist.go`, `internal/scaffold/repair.go`, `internal/commands/workitem/create.go`. |
| C4 | Registry locking with in-lock reload | Sequence 06 task 03 complete; review iterate fixed stale confirmation binding in `43c4acc`; final gate passed. |
| C5 | Priority locking and stale-lock TOCTOU fix | Sequence 06 tasks 04-05 complete; final gate passed. |

## D. Commit Semantics and Dispatch

| # | Check | Evidence |
|---|---|---|
| D1 | Temp-index `--only` engine and N-4 regression tests | Sequence 07 task 01 complete; final gate passed. |
| D2 | Rename-safe staged-file parsing | Sequence 07 task 02 complete; final gate passed. |
| D3 | Scoped sync/ref commits preserve unrelated staged files | Sequence 07 task 03 complete; final gate passed. |
| D4 | Root commit scoping / refs guard and ErrNoChanges classification | Sequence 07 task 04 complete; final gate passed. |
| D5 | Fest commitkit mirror workitem exists | `camp workitem list --json` shows `design-fest-commitcampaignroot-scope-unscoped-commit-2026-06-15` (`WI-25121c`). |
| D6 | Just-dispatch argv and flow runner positionals | Sequence 08 tasks 01-02 complete; final gate passed. |
| D7 | Read paths do not write | Sequence 08 tasks 03-04 complete; final gate passed. |

## E. App Contracts

| # | Check | Evidence |
|---|---|---|
| E1 | `doctor --json` exits non-zero on failure | Sequence 09 task 01 complete; final gate passed. |
| E2 | `flow items --json` versioned, stderr errors, non-zero failure | Sequence 09 task 02 complete; final gate passed. |
| E3 | `workitem create` atomic/self-cleaning retry model | Sequence 09 task 03 complete; final gate passed. |
| E4 | Intent JSON contract and multi-type handling | Sequence 09 task 04 complete; docs updated; final gate passed. |
| E5 | Workitem v1alpha6 stage vocabulary/no-stage/omitempty | Sequence 09 task 05 complete; docs updated; final gate passed. |
| E6 | Campaign-relative path semantics plus `campaign_root` | Sequence 09 task 06 complete; final gate passed. |
| E7 | Manifest annotations and doctor read/fix split | Sequence 09 task 07 complete; final gate passed. |
| E8 | Version stamping plus `profile` in `version --json` | Sequence 09 task 08 complete; final gate passed. |
| E9 | Concepts standard error plus versioned JSON | Sequence 09 task 09 complete; final gate passed. |
| E10 | JSON contract docs and stability matrix current | `GOFLAGS=-buildvcs=false just docs` completed and produced no diff; docs rows were updated in sequence 09. |
| E11 | Festival-app coordination workitem exists | `camp workitem list --json` shows `design-festival-app-camp-contract-changes-2026-06-15` (`WI-3db3f8`). |

## F. Consistency and Cleanup

| # | Check | Evidence |
|---|---|---|
| F1 | README install path and flow channel decision | Sequence 10 task 05 complete; final docs run produced no diff. |
| F2 | `reg` alias single home and root usage behavior | Sequence 10 task 04 complete; final gate passed. |
| F3 | JSON flag standardization, exit-code table, signal context | Sequence 10 tasks 01-03 complete; final gate passed. |
| F4 | fmt.Errorf ratchet active; query/commitkit migrated | `just gate` lint reported no new fmt.Errorf violators; sequence 11 task 01 complete. |
| F5 | Dead-code sweep before extraction | Sequence 11 tasks 02-03 complete in that order; final gate passed. |
| F6 | Shared statusdir primitive migrated | Sequence 11 task 05 complete; final gate passed. |
| F7 | Git plumbing, quest, intent, shell/nav, repair/TTY, small fixes | Sequences 10-12 complete; final gate passed. |
| F8 | Flaky-test hygiene and CLI docs | Sequence 12 tasks 01-02 complete; `GOFLAGS=-buildvcs=false just docs` produced no diff. |
| F9 | Dependency tags pinned | `go list -m github.com/lancekrogers/guild-scaffold github.com/Obedience-Corp/obey-shared` reported `v0.1.0` and `v0.4.4`. |

## G. Closeout

| # | Check | Evidence |
|---|---|---|
| G1 | Disposition log complete | `004_POLISH/plan/DISPOSITION_LOG.md` written. No data-loss/gate/boundary/app-contract findings are dispositioned as skipped; they are recorded as fixed. |
| G2 | Verified-negatives untouched | No intentional changes to the C-3 verified-negative list; any overlap was incidental to scoped refactors and covered by sequence gates. |
| G3 | FESTIVAL_GOAL checkboxes updated | `FESTIVAL_GOAL.md` checkboxes updated during closeout. |
| G4 | Completion report written | `004_POLISH/plan/COMPLETION_REPORT.md` written. |
| G5 | FE tag coverage | All CH0001 implementation commits carry `[OBEY-CAMPAIGN-8deed8b4-FE-CH0001]`. `bc2ad1f..HEAD` also contains five inherited shell commits from earlier branch history without the FE task suffix; these are not CH0001 task commits and were not rewritten. |

## Final Status

Final verification passed with documented closeout notes:

- `just gate`: passed.
- `just test integration`: passed, 373/373.
- Project worktree: clean and pushed.
- Draft PR: #324 at `5db31ec9753d1c44a3e6dd69229223405a16cfc4`.
