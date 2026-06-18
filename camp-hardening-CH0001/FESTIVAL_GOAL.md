---
fest_type: festival
fest_id: CH0001
fest_name: camp-hardening
fest_status: completed
fest_created: 2026-06-11T22:21:20.362498-06:00
fest_updated: 2026-06-15T17:01:33.516175-06:00
fest_tracking: true
---




# camp-hardening

**Status:** Planned | **Created:** 2026-06-11T22:21:20-06:00

## Festival Objective

**Primary Goal:** Remediate every finding from the fable-review-2026-06-10 camp review (recommendations R1-R36 plus the second-pass additions N-1 through N-27), eliminating the data-loss class first, then gate enforcement, fest-boundary violations, the atomic-write and locking sweep, agent-facing contract bugs, and finally the low-severity cleanup sweeps.

**Vision:** camp has no known code path that can destroy user data or fest-owned data: worktrees clean, cp --force, fresh/prune, project remove, sync fallback, migration renames, and intent moves are all guarded and regression-tested. The quality gates are green and trustworthy in both build profiles (stable and BUILD_TAGS=dev), enforced locally via just recipes and a pre-push hook with no hosted CI. Every agent-facing contract that festival-app shells (doctor --json, flow items --json, workitem list, intent JSON, stage filters, version) behaves as documented, and the remaining medium and low debt (fmt.Errorf, dead code, file splits, help text) is burned down.

## Success Criteria

### Functional Success

- [x] Data-loss class eliminated: `camp worktrees clean` falls through to `os.RemoveAll` only for confirmed-stale entries, never auto-removes `.git`-directory entries, prompts before deletion, and handles the empty `projectPath` branch safely (R2, second-pass correction 1)
- [x] `camp cp --force` rejects same-file copies (`os.SameFile`) and directory-into-itself copies, with container tests for both (N-1)
- [x] `camp fresh` and `camp project prune` skip dirty branch worktrees with `SkipReasonDirtyWorktree` and require an explicit flag to destroy dirty trees; `internal/prune` has table-driven unit tests plus container tests including `--remote-delete` (N-2, R16)
- [x] `camp project remove` refuses to delete `.git/modules/projects/<name>` while unpushed refs exist unless `--force`, naming the branches that would be lost (N-3)
- [x] `camp sync` stale-ref fallback never `os.RemoveAll`s a non-empty submodule dir without consent and restores via real submodule machinery, preserving `.git/modules` wiring (N-6); checkout errors surfaced and `validateUpdate` corrected (N-10)
- [x] Repair and workflow migrations are all-or-nothing: every (src exists, dst absent) pair pre-validated before any rename, failures exit non-zero with recovery guidance, and `commitRepairChanges` never runs after a failed migration (R4: INIT-1, WF-3, WF-4)
- [x] Intent `moveFile` carries the ErrFileExists guard for both CreateWithEditor and EditWithEditor call sites and the copy fallback fires only on EXDEV (R10, N-19); promote rollback removes only what it created (N-12)
- [x] Test gate green: manifest allowlist includes `workitem priority`; `go test ./...` passes with and without `-tags dev` (R1)
- [x] The five dead integration-tagged host test files are migrated into `tests/integration/` or deleted, and `go vet -tags=integration ./...` runs in `just lint` (R13, R14); container pool Reset covers `/root/.obey` and per-test work roots (R15)
- [x] Local gate enforcement exists for both profiles: `just test` runs dev-tagged tests, release recipes gate on both profiles, and a pre-push hook runs build, test, and lint locally; no GitHub Actions workflows are added or modified (R13, gate framing per second-pass section 4.7)
- [x] Fest-boundary regression tests exist for dungeon triage and the dungeon resolver: `festivals` and `projects` in the built-in exclusion map, resolver skips any `dungeon/` whose parent chain includes `festivals/` (R3: WF-1, WF-2)
- [x] No `os.WriteFile` remains on intent, quest, workflow, flow, state, gitignore-append, workitem private-clone, or config allowlist paths; all route through `fsutil.WriteFileAtomically` (R11: WF-6, N-21)
- [x] Registry and priority-store read-modify-write cycles run under `fsutil.AcquireFileLock` with re-load inside the lock; the fsutil stale-lock TOCTOU is fixed (R12: REG-1)
- [x] `cr` and `camp run` project just-dispatch exec `just` directly with argv (no `sh -c` flattening); flow runner passes extra args as positionals; tested with spaces, quotes, and metacharacters (R5: RUN-1)
- [x] Scoped-commit semantics fixed: `--only` worktree-content over-commit resolved (index-snapshot or loud divergence failure, N-4), staged-rename corruption fixed in `listStagedFiles` (N-5), then `syncParentRef`, `SyncSubmoduleRef`, and refs-sync commit with `Only` scoping (R6: GIT-2, GIT-7); root `camp commit` final commit scoped or refs-guarded (N-14); fest-side mirror of the commitkit scoping captured as a fest workitem (second-pass correction 2)
- [x] `camp pull all` never aborts a rebase it did not start (R7: GIT-3); `camp doctor --json` exits non-zero on failing checks (R8: UX-11)
- [x] Reads do not write: workitem list prune side effect, intent EnsureDirectories migration on read, and the status-all cache write are all moved to explicit write paths (R18, N-8, N-18)
- [x] Agent and app contract fixes land: `flow items --json` emits versioned JSON with errors to stderr and non-zero exit (N-7), `workitem create` is atomic or cleans up on failure (N-9), intent JSON contract is versioned (R20), stage filter no longer hides design/explore items and the stage vocabulary is published (R26), `intent list --type` honors all values (N-15), path semantics unified to campaign-relative plus `campaign_root` (N-16), manifest annotates every shipped command (N-11), version stamping with a `Profile` field in `camp version --json` (R21, N-25)
- [x] README install path corrected and the flow channel contradiction in scaffolded skills decided and fixed (R9: BR-6, BR-7); `reg` alias collision removed and `SilenceUsage` set on root (R17)
- [x] Cleanup sweeps complete: JSON flag standardization and error envelope (R19), exit-code table (R22), signal-aware root context (R23), fmt.Errorf forbidigo ratchet (R24), shared statusdir move primitive (R25), shell wrapper and nav fixes (R27), repair quiet failures and TTY guards (R28), dead code sweep before cmd-logic extraction (R29 then R30), git plumbing convergence (R31), quest robustness (R32), intent parser hardening (R33), flaky-test hygiene (R34), help and docs polish (R35), misc small fixes (R36)

### Quality Success

- [x] `just build`, `just test`, and `just lint` pass in BOTH profiles (stable and `BUILD_TAGS=dev`) at every implementation gate, verified by running both, not one
- [x] Every behavioral fix in the data-loss, fest-boundary, and contract classes ships with a regression test; destructive-path fixes (worktrees clean, cp, prune, project remove, sync fallback) get container integration tests
- [x] Zero `os.WriteFile` call sites remain on the state paths listed in R11/N-21, verified by grep at the final gate
- [x] `go vet -tags=integration ./...` is part of `just lint` and passes, so tagged code can no longer rot invisibly
- [x] No new GitHub Actions workflows exist in the diff; all enforcement is just recipes and pre-push hooks

## Progress Tracking

### Phase Completion

- [x] 001_INGEST: structure the review corpus (README, recommendations R1-R36, second-pass N-1 to N-27, seven findings docs) into purpose, requirements, constraints, and context specs
- [x] 002_PLAN: design the implementation phase breakdown, sequencing, and dependency order (data loss first, then gates, boundary, atomicity, contracts, cleanup)
- [x] IMPLEMENT: execute the remediation sequences in the linked worktree with quality gates per sequence
- [x] POLISH: verify the full success-criteria checklist, close the low-severity sweeps, and confirm both-profile gates end to end

## Complete When

- [x] All phases completed
- [x] All P0/P1 findings (R1-R18 plus N-1, N-2, N-3, N-6) are fixed with regression tests, and all P2/P3 items (R19-R36 plus remaining N findings) are fixed or explicitly dispositioned in the decision log
- [x] `go test ./...` passes with and without `-tags dev` on the camp-hardening branch, and `go vet -tags=integration ./...` is clean
- [x] Local gate enforcement (both-profile just recipes plus pre-push hook) is in place with no GitHub Actions changes