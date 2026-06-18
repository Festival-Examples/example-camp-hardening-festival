# Festival Structure: camp-hardening (CH0001)

## Festival Goal

Remediate every finding from the fable-review-2026-06-10 camp review (R1-R36 plus N-1 through N-27): eliminate the data-loss class first, then gate enforcement, fest-boundary violations, the atomic-write and locking sweep, agent-facing contract bugs, and finally the low-severity cleanup sweeps. All gates are local (just recipes plus a pre-push hook; GitHub Actions stays untouched) and run in both build profiles.

## Hierarchy

- **Festival:** camp-hardening (CH0001)
  - **Phase 001_INGEST** (ingest) — COMPLETE: review corpus structured into purpose/requirements/constraints/context
  - **Phase 002_PLAN** (planning) — THIS PHASE: plan, decisions, scaffold
  - **Phase 003_IMPLEMENT** (implementation) — all remediation work, 12 sequences in dependency order
    - **Sequence 01_gate_restoration** — make the test gate green so every later sequence's quality gates are meaningful (R1)
      - 01_fix_manifest_allowlist (R1 / TEST-1)
      - 02_baseline_gate_survey (record both-profile build/test/lint baseline; no fixes)
    - **Sequence 02_destructive_command_safety** — data-loss class, part 1: commands that delete user files (R2, N-1, N-2, R16, N-3)
      - 01_worktrees_clean_guards (R2 / GIT-1 plus second-pass correction 1; container tests)
      - 02_cp_copy_guards (N-1: SameFile and dir-into-itself; container tests)
      - 03_prune_dirty_worktree_protection (N-2: branch-worktree clean check, explicit destroy flag, fresh force)
      - 04_prune_test_suite (R16 / TEST-4: fake-executor unit tests plus container tests incl --remote-delete)
      - 05_project_remove_unpushed_refs_guard (N-3; container test)
    - **Sequence 03_sync_and_migration_safety** — data-loss class, part 2: sync/pull and migration clobber paths (N-6, N-10, R7, R4, R10, N-19, N-12)
      - 01_sync_fallback_quarantine (N-6: no RemoveAll without consent; real submodule machinery)
      - 02_sync_checkout_drift_validation (N-10: surface checkout errors; fix validateUpdate)
      - 03_pull_all_rebase_guard (R7 / GIT-3; container test with pre-existing rebase, fixture shared with 01)
      - 04_repair_migration_all_or_nothing (R4: WF-3 pre-validation plus INIT-1 no-auto-commit)
      - 05_workflow_migrate_collision_check (R4: WF-4)
      - 06_intent_movefile_and_promote_rollback (R10 / WF-5, N-19 EXDEV gate, N-12 rollback scope)
    - **Sequence 04_gate_enforcement** — trustworthy local gates in both profiles before the broad sweeps (R13, R14, R15)
      - 01_test_runner_build_tags (BR-1: tasks.Test honors BUILD_TAGS; just test dev)
      - 02_release_recipe_gating (BR-2: tag only after both-profile tests)
      - 03_prepush_hook (DEC-005; no GitHub Actions)
      - 04_integration_tagged_test_migration (R14 / TEST-2: migrate or delete the 5 dead files; vet -tags=integration in just lint)
      - 05_container_reset_paths (R15 / TEST-3)
    - **Sequence 05_fest_boundary** — camp must never touch fest-owned state (R3, WF-7)
      - 01_dungeon_exclusion_and_resolver_guards (WF-1, WF-2; regression tests; default .crawl.yaml)
      - 02_promote_ingest_boundary (WF-7: stop MkdirAll into fest layout; surface fest stderr; fest-ingest capability workitem)
    - **Sequence 06_atomicity_and_locking** — durability sweep over the existing fsutil primitives (R11, N-21, R12)
      - 01_intent_quest_atomic_writes (WF-6: intent service/rename/notes, quest)
      - 02_workflow_flow_state_config_atomic_writes (WF-6 rest: workflow, flow registry, state plus LoadHistory, gitignore append, allowlist N-21, delete private clone)
      - 03_registry_update_lock (REG-1: config.UpdateRegistry, migrate mutating callers)
      - 04_priority_store_lock (priority.WithLock incl TUI path)
      - 05_fsutil_stale_lock_toctou (REG-2)
    - **Sequence 07_commit_semantics** — the scoped-commit engine chain, in mandated order (N-4, N-5, R6, N-14, N-23)
      - 01_only_index_snapshot (N-4 per DEC-001; lands FIRST)
      - 02_staged_rename_parsing (N-5: name-status -z in listStagedFiles)
      - 03_sync_commit_scoping (R6: syncParentRef, SyncSubmoduleRef, refs-sync GIT-7)
      - 04_root_commit_scope_and_errnochanges (N-14 plus N-23 classification)
      - 05_fest_commitkit_workitem (coordination workitem for fest's commit.go mirror; second-pass correction 2)
    - **Sequence 08_argv_and_read_paths** — argv integrity and reads-do-not-write (R5, R18, N-8, N-18)
      - 01_just_dispatch_argv (RUN-1: cmdutil plus run.go; metacharacter tests)
      - 02_flow_runner_positionals (flow runner arg splicing)
      - 03_workitem_list_read_only (R18: prune to explicit write paths)
      - 04_intent_read_migration_and_status_cache (N-8: EnsureDirectories off reads; N-18: status-all cache write and dead --no-cache)
    - **Sequence 09_app_contracts** — every surface festival-app shells, versioned and correct (R8, N-7, N-9, R20, R26, N-15, N-16, N-11, R21, N-25, N-17, N-24)
      - 01_doctor_json_exit (R8 / UX-11)
      - 02_flow_items_json (N-7: real versioned JSON, stderr errors, non-zero exit)
      - 03_workitem_create_atomicity (N-9: FA0011 retry compatibility)
      - 04_intent_json_contract (R20 / JSON-2 envelope; N-15 --type; N-24 -f collision and machine output; JSON-1 doc rows)
      - 05_stage_vocabulary (R26: LifecycleStage constants, explicit no-stage, per-type vocab in payload, ritual/chains; JSON-3 omitempty rides the schema bump)
      - 06_path_semantics_unification (N-16: campaign-relative plus campaign_root everywhere)
      - 07_manifest_annotations (N-11: annotate every shipped command; split doctor read vs --fix)
      - 08_version_stamping_and_profile (R21: BR-3, BR-4; N-25; UX-15 RunE)
      - 09_concepts_json (N-17)
    - **Sequence 10_cli_consistency** — flags, exit codes, signals, launch docs, shell/nav (R19, R22, R23, R17, R9, R27, R28, N-13)
      - 01_json_flag_standardization (R19: UX-7, UX-8, UX-21, UX-20)
      - 02_exit_code_table (R22: UX-13, UX-14)
      - 03_signal_context (R23: ERR-2, ERR-3, ERR-4)
      - 04_alias_and_usage_fixes (R17: UX-1 reg collision, UX-12 SilenceUsage; UX-6 dead global flags)
      - 05_readme_and_flow_channel (R9 per DEC-002: BR-6, BR-7)
      - 06_shell_and_nav_fixes (R27: SH-1, SH-2, NAV-1, NAV-2, NAV-3)
      - 07_repair_failures_and_tty_guards (R28: INIT-2, ERR-5, UX-19)
      - 08_pull_push_flag_handling (N-13)
    - **Sequence 11_debt_burn_down** — structural debt in mandated internal order (R24, R29 before R30, R31, R25, R32, R33, N-20, N-22, N-26, N-27)
      - 01_fmt_errorf_ratchet (R24 / ERR-1: forbidigo plus allowlist; query.go and commitkit first)
      - 02_dead_code_sweep (R29 / ARCH-5; STRICTLY before 03)
      - 03_cmd_logic_extraction (R30 / ARCH-1, ARCH-2: Tree B canonical, extractions)
      - 04_git_plumbing_convergence (R31: GIT-4, GIT-5, GIT-8, GIT-10, LC_ALL per ERR-8)
      - 05_statusdir_move_primitive (R25 per DEC-003; migrates the four implementations; fixes dungeon TOCTOU/stat-conflation)
      - 06_quest_robustness (R32: skip-and-warn List, Mkdir-claim slugs, BR-5 stable surface story)
      - 07_intent_parser_hardening (R33 plus N-26: line-anchored delimiters, unknown-key round-trip, O_EXCL, removeAllCopies id check, gather failures)
      - 08_small_safety_fixes (N-20 skills force symlinks, N-22 pins symlinked root, N-27 buildutil clean guard)
    - **Sequence 12_test_hygiene_and_docs** — suite hygiene and the polish batch (R34, R35, R36, BR-8)
      - 01_flaky_test_fixes (R34: TEST-6, TEST-8: event-driven locks, Chtimes, single Reset, t.Setenv/t.Chdir, allowlist burn-down start)
      - 02_help_docs_polish (R35: UX-2, UX-3, UX-5, UX-10; BR-8 regen; flow help contradictions)
      - 03_misc_small_fixes (R36: GIT-6 amend, GIT-9 lock age, LINK-1, LINK-2, dungeon hint, ARCH-8 pathsafe merge, shared slug package, ID-scheme matrix)
      - 04_upstream_dependency_tags (ARCH-9: tag guild-scaffold and obey-shared, pin tags)
  - **Phase 004_POLISH** (planning) — verification and closeout only (DEC-006)
    - Final verification checklist mapping every FESTIVAL_GOAL success criterion to a concrete command: both-profile just build/test/lint end to end, `go vet -tags=integration ./...`, residual `os.WriteFile` grep on the R11/N-21 paths, no-`.github/workflows`-diff check, decision-log disposition review for every P2/P3 item, regression-test presence audit for the data-loss/boundary/contract classes.

## As-built scaffold summary

Scaffolded and validated (fest validate 100/100) at the end of 002_PLAN. Every implementation sequence carries the four auto-appended quality gates (testing, review, iterate, fest-commit) after its implementation tasks; the testing gate is tailored to camp's local both-profile commands (`just build stable`, `just build dev`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `just vet`, `just test integration`). Gates are local only; GitHub Actions stays untouched.

| Sequence | Implementation tasks | Gate tasks |
|---|---|---|
| 01_gate_restoration | 2 | 4 |
| 02_destructive_command_safety | 5 | 4 |
| 03_sync_and_migration_safety | 6 | 4 |
| 04_gate_enforcement | 5 | 4 |
| 05_fest_boundary | 2 | 4 |
| 06_atomicity_and_locking | 5 | 4 |
| 07_commit_semantics | 5 | 4 |
| 08_argv_and_read_paths | 4 | 4 |
| 09_app_contracts | 9 | 4 |
| 10_cli_consistency | 8 | 4 |
| 11_debt_burn_down | 8 | 4 |
| 12_test_hygiene_and_docs | 4 | 4 |
| **Total** | **63** | **48** |

Note: fest.yaml's default `*_docs` exclusion pattern was removed so 12_test_hygiene_and_docs (a real implementation sequence) receives quality gates like every other sequence. 004_POLISH is a planning-type verification phase (DEC-006) with freeform docs (VERIFICATION_CHECKLIST.md scaffolded in its plan/); it has no sequences or gates by design.

## Finding-to-sequence map (completeness check)

| Findings | Home |
|---|---|
| R1 (TEST-1) | 01.01 |
| R2 (GIT-1, correction 1) | 02.01 |
| N-1 | 02.02 |
| N-2 | 02.03 |
| R16 (TEST-4 prune) | 02.04 |
| N-3 | 02.05 |
| N-6 | 03.01 |
| N-10 | 03.02 |
| R7 (GIT-3) | 03.03 |
| R4 (INIT-1, WF-3) | 03.04 |
| R4 (WF-4) | 03.05 |
| R10 (WF-5), N-19, N-12 | 03.06 |
| R13 (BR-1, BR-2, local framing) | 04.01-04.03 |
| R14 (TEST-2) | 04.04 |
| R15 (TEST-3) | 04.05 |
| R3 (WF-1, WF-2) | 05.01 |
| WF-7 | 05.02 |
| R11 (WF-6), N-21 | 06.01-06.02 |
| R12 (REG-1, priority lock) | 06.03-06.04 |
| REG-2 | 06.05 |
| N-4 | 07.01 |
| N-5 | 07.02 |
| R6 (GIT-2, GIT-7) | 07.03 |
| N-14, N-23 | 07.04 |
| Fest commitkit mirror (correction 2) | 07.05 |
| R5 (RUN-1) | 08.01-08.02 |
| R18 | 08.03 |
| N-8, N-18 | 08.04 |
| R8 (UX-11) | 09.01 |
| N-7 | 09.02 |
| N-9 | 09.03 |
| R20 (JSON-2, JSON-1), N-15, N-24 | 09.04 |
| R26, JSON-3 | 09.05 |
| N-16 | 09.06 |
| N-11 | 09.07 |
| R21 (BR-3, BR-4), N-25, UX-15, JSON-4, JSON-5 | 09.08 |
| N-17 | 09.09 |
| R19 (UX-7, UX-8, UX-20, UX-21) | 10.01 |
| R22 (UX-13, UX-14) | 10.02 |
| R23 (ERR-2, ERR-3, ERR-4) | 10.03 |
| R17 (UX-1, UX-12), UX-6 | 10.04 |
| R9 (BR-6, BR-7) | 10.05 |
| R27 (SH-1, SH-2, NAV-1, NAV-2, NAV-3) | 10.06 |
| R28 (INIT-2, ERR-5, UX-19) | 10.07 |
| N-13 | 10.08 |
| R24 (ERR-1) | 11.01 |
| R29 (ARCH-5) | 11.02 |
| R30 (ARCH-1, ARCH-2) | 11.03 |
| R31 (GIT-4, GIT-5, GIT-8, GIT-10, ERR-8) | 11.04 |
| R25 (statusdir unification, dungeon TOCTOU) | 11.05 |
| R32 (quest, BR-5) | 11.06 |
| R33, N-26 | 11.07 |
| N-20, N-22, N-27 | 11.08 |
| R34 (TEST-6, TEST-8) | 12.01 |
| R35 (UX-2, UX-3, UX-5, UX-10, BR-8) | 12.02 |
| R36 (GIT-6, GIT-9, LINK-1, LINK-2, ARCH-8, slug, ID matrix, hints) | 12.03 |
| ARCH-9 | 12.04 |
| Final verification (grep gates, no-CI check, dispositions) | 004_POLISH |

Smaller findings folded into their parent's task: SH-3 and NAV-5 (latent hygiene, noted in 10.06 as in-scope-if-touched), ERR-6 Get-prefix deletions (ride 11.02 dead-code sweep), ERR-7 magic strings (ride 09.05 stage constants and 11.05 primitive), ARCH-3 naming inversion and ARCH-6 file splits (ride 11.03 extraction targets), ARCH-7 ui leakage (ride 11.03), REG-3 state file (ride 06.02), BR-5 (ride 11.06), BR-9 release-notes nits (ride 12.02), JSON-6 matrix (documentation outcome of 09.x). Nothing in R1-R36 or N-1..N-27 lacks a home.

## Dependencies

| Item | Blocked by | Rationale |
|---|---|---|
| Every sequence after 01 | 01_gate_restoration | The test gate is red on main; per-sequence quality gates are meaningless until R1 lands. D007 records this scope-capped bootstrap exception to the data-loss-first ordering |
| 04_gate_enforcement | 02, 03 | FESTIVAL_GOAL orders data loss before gate enforcement (D007); 04 then makes both-profile gates trustworthy for everything after |
| 05 through 12 | 04_gate_enforcement | Broad sweeps need dev-profile tests and vet -tags=integration actually running |
| 07.03 (R6) | 07.01 (N-4), 07.02 (N-5) | Second-pass correction 3: do not extend the Only pattern until its semantics are fixed |
| 11.03 (R30) | 11.02 (R29) | Recommendations: dead code deleted strictly before extraction so nothing dead is migrated |
| 11.05 (R25) | 03.04-03.06 | DEC-003: targeted P0 migration fixes land first; the primitive consolidates them later |
| 09.05 schema bump | 09.04 | Intent contract and stage vocabulary changes coordinate in docs/json-contracts.md updates |
| 004_POLISH | all of 003 | Verification-only phase (DEC-006) |

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| File:line drift between review baselines (5c22d2b/11c270f) and the worktree branch point (bc2ad1f) | Medium | Medium | Every task doc verifies targets against the worktree during planning (GAP-7); tasks instruct re-location by symbol |
| N-4 fix destabilizes every auto-commit call site | Medium | High | DEC-001 chooses the approach; 07.01 ships with scratch-repo-verified semantics tests before 07.03 builds on it |
| commitkit scoping change breaks fest | Medium | High | Treated as a public API change (C-5); fest workitem 07.05 communicates it; compat notes in the change |
| Schema bumps break festival-app parsing | Medium | High | DEC-004: versioned bumps plus doc rows plus an app coordination workitem; never silent shape changes |
| Sequence ends with a red gate in one profile | Medium | High | Every sequence's verification plan runs BOTH profiles explicitly; the auto-appended testing gate enforces it |
| Dead-code deletion removes something a later extraction needs | Low | Medium | R29 task requires deadcode-tool verification against both tag sets before deleting |
| 12.04 upstream tagging blocked by repo access | Low | Low | Task includes a disposition path: record as dispositioned with the pinned pseudo-versions documented |
