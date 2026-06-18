# Implementation Plan: camp-hardening (CH0001)

## Overview

Remediate all R1-R36 and N-1..N-27 findings in the linked worktree `projects/worktrees/camp/camp-hardening` (branch point `bc2ad1f`, which includes PR #318 `workitem priority`; review citations are from `5c22d2b`/`11c270f` and every task verifies its file:line targets against the worktree). Two phases follow this plan: 003_IMPLEMENT (12 sequences, all code work) and 004_POLISH (verification-only closeout, D006). Quality gates (testing, review, iterate, fest-commit) auto-append to every implementation sequence per fest.yaml; sequence tasks do not duplicate them.

**Standing verification commands** (the "both-profile gate", referenced by every sequence; from the worktree root):

```bash
just build                       # stable profile build
BUILD_TAGS=dev just build        # dev profile build
just test unit                   # stable unit suite
BUILD_TAGS=dev just test unit    # dev unit suite (recipe created in sequence 04; until then: go test -tags dev ./...)
just lint                        # stable lint incl. go vet; gains -tags=integration vet in sequence 04
BUILD_TAGS=dev just lint         # dev-profile lint/vet (until sequence 04 wires it: go vet -tags dev ./...)
just test integration            # container suite, for sequences adding container tests
```

Per C-2, "both-profile gate" in every sequence verification plan below means ALL THREE of build, test, and lint executed in BOTH profiles (stable and `BUILD_TAGS=dev`), not a subset.

Until sequence 04 lands, dev-profile tests run via `go test -tags dev -count=1 -short ./...` directly. After sequence 04, `just gate` (D005) is the closing command for every sequence.

## Phase Breakdown

### Phase 003_IMPLEMENT (implementation)

#### Sequence 01_gate_restoration

- **Goal:** the unit gate is green in both profiles so every later sequence's quality gates are meaningful (R1/TEST-1).
- **Tasks:**
  - [ ] 01_fix_manifest_allowlist: add `workitem priority` to the manifest allowlist, bump the restricted-command counts (stable 27->28, dev 43->44 expectations)
  - [ ] 02_baseline_gate_survey: run the full standing matrix in both profiles, record pass/fail baseline in the task notes; NO fixes beyond R1
- **Verification:** `go test -count=1 -short -run TestManifestCommand ./cmd/camp` green in both profiles; full `go test ./...` and `go test -tags dev ./...` green; `just build` both profiles.

#### Sequence 02_destructive_command_safety

- **Goal:** no camp command that deletes files can destroy uncommitted or untracked user work without explicit consent (R2+correction1, N-1, N-2, R16, N-3).
- **Tasks:**
  - [ ] 01_worktrees_clean_guards: RemoveAll only for gitdir-target-missing staleness; never auto-remove `.git`-directory entries; confirmation prompt; fix the empty-projectPath bypass; `--force` plus dirty check for everything else; container tests (package has zero today)
  - [ ] 02_cp_copy_guards: `os.SameFile` rejection, dest-under-src rejection for dirs, cp(1)-style errors; container tests for same-file truncation and self-recursion
  - [ ] 03_prune_dirty_worktree_protection: branch-worktree clean check mirroring `detachedWorktreeClean`, `SkipReasonDirtyWorktree`, explicit destroy flag; `fresh` stops hardcoding `Force: true` past the dirty check
  - [ ] 04_prune_test_suite: table-driven unit tests (fake executor) for candidate/forced/merged classification; container tests for `camp project prune` incl `--remote-delete` against a bare origin
  - [ ] 05_project_remove_unpushed_refs_guard: refuse `.git/modules` deletion while unpushed refs exist unless `--force`, naming the branches; fix the "files remain in place" help text; container test
- **Verification:** both-profile gate; `just test integration` with the new worktrees/cp/prune/remove tests; manual `--dry-run` spot checks listed per task.

#### Sequence 03_sync_and_migration_safety

- **Goal:** sync/pull and every migration/move path is all-or-nothing and never silently destroys or clobbers (N-6, N-10, R7, R4, R10, N-19, N-12).
- **Tasks:**
  - [ ] 01_sync_fallback_quarantine: no RemoveAll of non-empty submodule dirs without consent (quarantine-rename); fallback via real submodule machinery preserving `.git/modules`; relative URL resolution against the superproject remote
  - [ ] 02_sync_checkout_drift_validation: surface `CheckoutDefaultBranch` errors; fix `validateUpdate`'s backwards drift skip; warn or fast-forward on branch-tip != recorded-gitlink
  - [ ] 03_pull_all_rebase_guard: probe `IsRebaseInProgress` pre-pull, skip with warning; abort only self-initiated rebases; container test with a pre-existing conflicted rebase (hostile-remote fixture shared with 01)
  - [ ] 04_repair_migration_all_or_nothing: pre-validate every (src exists, dst absent) pair via `statuspath.ExistingItemPath`; non-zero exit with recovery guidance; `commitRepairChanges` never after failure
  - [ ] 05_workflow_migrate_collision_check: MigrateV1ToV2 destination checks plus per-item error collection; v2 schema write atomic
  - [ ] 06_intent_movefile_and_promote_rollback: ErrFileExists guard inside `moveFile` (covers CreateWithEditor and EditWithEditor), copy fallback gated on EXDEV; promote rollback Mkdir-tracked, removes only what it created
- **Verification:** both-profile gate; container tests for the sync fallback and rebase guard; unit tests for migration pre-validation failure paths asserting non-zero exit and no auto-commit. Tasks structure fixes as small no-replace helpers for D003's later extraction.

#### Sequence 04_gate_enforcement

- **Goal:** trustworthy local enforcement in both profiles; no hosted CI (R13 local framing, R14, R15; D005).
- **Tasks:**
  - [ ] 01_test_runner_build_tags: `tasks.Test` honors `BUILD_TAGS`; `just test dev` (or `BUILD_TAGS=dev just test`) recipes; `just gate-fast` and `just gate` recipes per D005
  - [ ] 02_release_recipe_gating: dev/rc/stable/promote recipes run `just gate` before tagging; document the festival-repo division of labor
  - [ ] 03_prepush_hook: committed `.githooks/pre-push` exec-ing `just gate-fast` (CAMP_GATE_FULL=1 upgrade); `just hooks install` sets core.hooksPath; contributor docs
  - [ ] 04_integration_tagged_test_migration: migrate worthwhile coverage from the five dead `//go:build integration` host files into `tests/integration/`, delete the rest (commit one does not compile); add `go vet -tags=integration ./...` to `just lint`
  - [ ] 05_container_reset_paths: Reset rm list gains `/root/.obey` and per-test work roots, drops legacy `/root/.config/camp`; remove the settings_test manual workaround
- **Verification:** `just gate` green end to end; `go vet -tags=integration ./...` clean and wired into lint; pre-push hook fires on a scratch push; ZERO files under `.github/workflows/` in the diff.

#### Sequence 05_fest_boundary

- **Goal:** camp can never relocate, adopt, or write into fest-owned state (R3: WF-1, WF-2; WF-7).
- **Tasks:**
  - [ ] 01_dungeon_exclusion_and_resolver_guards: `festivals` and `projects` in the built-in exclusion map; resolver skips any `dungeon/` with `festivals/` in its parent chain; regression tests for both; default `dungeon/.crawl.yaml` scaffolded at init
  - [ ] 02_promote_ingest_boundary: stop MkdirAll-ing fest-internal ingest dirs (skip the copy when absent); surface fest CLI stderr on promote failure; create the fest workitem for a `fest ingest <file>` capability
- **Verification:** both-profile gate; regression tests proving `camp dungeon move festivals archived` refuses and the resolver ignores `festivals/dungeon/`.

#### Sequence 06_atomicity_and_locking

- **Goal:** every state write is atomic and every read-modify-write is locked (R11+N-21, R12: REG-1, REG-2).
- **Tasks:**
  - [ ] 01_intent_quest_atomic_writes: intent `service.go:120,292,354`-class sites, `rename.go`, `notes.go`; quest `quest.go:271` -> `fsutil.WriteFileAtomically`
  - [ ] 02_workflow_flow_state_config_atomic_writes: workflow `service_init.go`/`service_sync_migrate.go`, flow `registry.go:76`, state `location.go:105` (plus `LoadHistory` skipping corrupt lines, REG-3), gitignore append in repair, config `allowlist.go:84` (N-21); delete the workitem private `atomicWriteFile` clone
  - [ ] 03_registry_update_lock: `config.UpdateRegistry(ctx, mutate)` over `fsutil.AcquireFileLock`, re-load inside the lock; migrate register/clone/scaffold/switch mutating callers
  - [ ] 04_priority_store_lock: `priority.WithLock` mirroring `links.WithLock`; CLI and TUI paths
  - [ ] 05_fsutil_stale_lock_toctou: rename-steal (or PID-liveness) so only one stealer wins; REG-2's 30s assumption documented or made configurable
- **Verification:** both-profile gate; `grep -rn "os.WriteFile" internal/intent internal/quest internal/workflow internal/flow internal/state internal/config internal/scaffold internal/commands/workitem` shows only non-state or test sites (exact expected-remainder list recorded in the task); concurrency regression tests for registry and priority store.

#### Sequence 07_commit_semantics

- **Goal:** scoped commits commit exactly what they claim, in the mandated order N-4 -> N-5 -> R6 -> N-14 (D001; second-pass corrections 2 and 3).
- **Tasks:**
  - [ ] 01_only_index_snapshot: temp `GIT_INDEX_FILE` engine per D001; staged-blob semantics for `--staged`; scratch-repo-style regression tests reproducing the v2-staged/v3-worktree case
  - [ ] 02_staged_rename_parsing: `listStagedFiles` parses `--name-status -z` with both rename sides (reuse the ExpandTrackedPaths parser); regression test for the staged `git mv` corruption
  - [ ] 03_sync_commit_scoping: `syncParentRef` and commitkit `SyncSubmoduleRef` commit scoped to the submodule path; whole-repo `HasStagedChanges` no-op guard fixed; refs-sync `Only: toSync` plus printed skips (GIT-7)
  - [ ] 04_root_commit_scope_and_errnochanges: root `camp commit` final commit scoped to camp-staged paths or refuses staged `projects/*` gitlinks without `--include-refs` (N-14); "no changes added to commit" classified ErrNoChanges with the refs-excluded hint (N-23)
  - [ ] 05_fest_commitkit_workitem: `camp workitem create --type design` for the fest project documenting fest's `commit.go:541-553` unscoped reuse, the new commitkit semantics, and the required mirror change; references D001
- **Verification:** both-profile gate; the N-4/N-5 regression tests; a container test asserting pre-staged unrelated files survive a sync commit untouched; commitkit public API compile-compat noted for fest.

#### Sequence 08_argv_and_read_paths

- **Goal:** argv survives dispatch byte-for-byte and read commands never write (R5, R18, N-8, N-18).
- **Tasks:**
  - [ ] 01_just_dispatch_argv: project dispatch execs `just` with argv and `cmd.Dir`; `sh -c` only for the documented raw form; run.go's other `strings.Join` flattenings fixed; tests with spaces/quotes/`$`/`;`/globs
  - [ ] 02_flow_runner_positionals: `sh -c '<cmd> "$@"' sh args...`; same metacharacter tests
  - [ ] 03_workitem_list_read_only: `priority.Prune` moved to priority command, TUI mutations, and `workitem doctor --fix`; list (incl `--json`) writes nothing
  - [ ] 04_intent_read_migration_and_status_cache: intent list/find/count/show stop calling `EnsureDirectories`; migration runs in `camp init --repair` and write paths, reads warn-only (N-8); status-all cache write removed or made an explicit honored flag with a real read path (N-18)
- **Verification:** both-profile gate; `cr just <recipe> -m "two words"`-shaped tests; a read-only assertion test (run list/read commands in a fixture, assert zero git status change).

#### Sequence 09_app_contracts

- **Goal:** every surface festival-app shells is versioned, correct, and preflightable (R8, N-7, N-9, R20, R26, N-15, N-16, N-11, R21, N-25, N-17, N-24; D002, D004).
- **Tasks:**
  - [ ] 01_doctor_json_exit: exit code applied after JSON output; test asserting non-zero under `--json` with failing checks
  - [ ] 02_flow_items_json: versioned payload via jsoncontract; errors to stderr, non-zero exit; `cmd.Context()`; dead HistoryOptions.JSON resolved
  - [ ] 03_workitem_create_atomicity: derive id/ref before MkdirAll or clean up on failure; retry-after-failure test (FA0011 model)
  - [ ] 04_intent_json_contract: `intents/v1alpha1` envelope on list/find/count/show; `--json` with error envelope; `--format json` deprecated alias; multi `--type` honored (N-15); `intent add` `-f` collision and machine-readable created id/path (N-24); docs rows incl `workitem priority` (JSON-1)
  - [ ] 05_stage_vocabulary: LifecycleStage constants, explicit no-stage, per-type vocab in payload, ritual/chains discovery; workitems v1alpha6 bump folding JSON-3 omitempty fix; festival-app coordination workitem per D004
  - [ ] 06_path_semantics_unification: campaign-relative paths plus `campaign_root` across workitem/intent/quest/status-all (N-16)
  - [ ] 07_manifest_annotations: every shipped command annotated; doctor read vs `--fix` split; manifest version bumped
  - [ ] 08_version_stamping_and_profile: ldflags threaded into cross-platform recipes; ReadBuildInfo fallback; `Profile` build-tag constant in `camp version --json`; `--short`/`--json` precedence and RunE conversion (N-25, UX-15)
  - [ ] 09_concepts_json: standard not-in-campaign error; versioned `--json` (N-17)
- **Verification:** both-profile gate; contract tests asserting schema_version strings; `docs/json-contracts.md` updated rows; app coordination workitem exists.

#### Sequence 10_cli_consistency

- **Goal:** one flag idiom, one exit-code table, signal-aware context, honest help and launch docs (R19, R22, R23, R17, R9, R27, R28, N-13; D002).
- **Tasks:**
  - [ ] 01_json_flag_standardization: `--json` aliased on `--format` commands; `flow add --json` -> `--from-json`; remaining `--json` commands wrapped in jsoncontract.RunE; status-all empty case emits versioned JSON (UX-20)
  - [ ] 02_exit_code_table: CLI-wide table (0/1/2/3+ documented); doctor/sync/clone aligned; os.Exit sites replaced with typed CommandError (UX-14)
  - [ ] 03_signal_context: `signal.NotifyContext` plus `ExecuteContext`; nil-ctx fallbacks deleted; ctx-aware retry backoff (ERR-3); helper-context cleanup where reachable (ERR-4)
  - [ ] 04_alias_and_usage_fixes: `reg` only on registry plus no-duplicate-aliases test (UX-1); root `SilenceUsage` with per-command patches removed, flag-error usage retained (UX-12); dead `--config`/`--verbose` globals wired or deleted (UX-6)
  - [ ] 05_readme_and_flow_channel: README install path (BR-7); channel-aware scaffolded skills/templates, README flow caveat, dungeon-add help (BR-6 per D002); docs recipe BUILD_TAGS pin (BR-5 docs leak)
  - [ ] 06_shell_and_nav_fixes: wrappers drop `2>/dev/null`, check exit codes (SH-1); go/switch `--print` passthrough (SH-2); nav stat-before-print with one forced rebuild (NAV-1); `--list`+`--print` rejected, disambiguation to stderr under --print (NAV-2); locked index rebuild with surfaced cache errors (NAV-3)
  - [ ] 07_repair_failures_and_tty_guards: init registry-save failure returns error, fest-init failure prints cause, swallowed cleanups warn (INIT-2, ERR-5); IsTerminal guards plus agent_allowed annotations on intent explore, settings, intent crawl, dungeon crawl (UX-19)
  - [ ] 08_pull_push_flag_handling: intercept only `--project`/`--project=`, honor `--`, fix the `-p <display-name>` hint (N-13)
- **Verification:** both-profile gate; help-screen assertions; exit-code table documented and tested; live wrapper smoke per task.

#### Sequence 11_debt_burn_down

- **Goal:** structural debt removed in safe order: ratchet, dead code, extraction, plumbing, unification, subsystem hardening (R24, R29 strictly before R30, R31, R25 per D003, R32, R33, N-20, N-22, N-26, N-27).
- **Tasks:**
  - [ ] 01_fmt_errorf_ratchet: forbidigo rule banning fmt.Errorf outside tools/ with burn-down allowlist; `internal/git/query.go` and `pkg/commitkit` migrated first (ERR-1)
  - [ ] 02_dead_code_sweep: delete the ARCH-5 inventory (~216 symbols: clone validation, concept FSService, nav query/picker, shortcuts Expander/FeedbackWriter, sync preflights, git/config/errors stragglers, crawl fake to _test); deadcode-tool verification against BOTH tag sets before and after; ERR-6 Get-prefix deletions ride along
  - [ ] 03_cmd_logic_extraction: Tree B constructor style documented canonical; pull/status_all/shortcuts/leverage/dungeon-crawl/go.go logic extracted into internal/; `EnsureRefForCommit` into internal/workitem; run.go duplicated resolution consolidated; ARCH-6 file-size and ARCH-7 ui-leakage targets addressed where touched
  - [ ] 04_git_plumbing_convergence: status-all `-z` porcelain parser (GIT-4); `FilterTracked` `-z` (GIT-5); converge on RunGitCmd; literal exclusion pathspecs; `LC_ALL=C` on git execs (ERR-8); gitdir-target worktree-vs-submodule detection (GIT-8)
  - [ ] 05_statusdir_move_primitive: shared package per D003 (no-replace, dated buckets, EXDEV fallback); four implementations migrated; dungeon TOCTOU and stat-conflation fixed; sequence-03 regression tests as the conformance baseline
  - [ ] 06_quest_robustness: List skip-and-warn; Mkdir-claim slug reservation; quest ID uniqueness; stable-build quest surface channel-aware (BR-5) per D002's mechanism
  - [ ] 07_intent_parser_hardening: line-anchored delimiters; unknown-key round-trip; O_EXCL guarded writes; removeAllCopies id verification; gather archive failures surfaced plus the N-26 orphan cleanup
  - [ ] 08_small_safety_fixes: skills `--force` unmanaged-symlink protection (N-20); pins symlinked-root migration fix (N-22); buildutil clean cwd guard (N-27)
- **Verification:** both-profile gate after EVERY task (this sequence has the highest regression surface); deadcode tool re-run; forbidigo active in `just lint`.

#### Sequence 12_test_hygiene_and_docs

- **Goal:** suite reliability and the documentation/polish batch (R34, R35, R36, ARCH-9, BR-8).
- **Tasks:**
  - [ ] 01_flaky_test_fixes: event-driven lock-release tests; `os.Chtimes` over mtime sleeps; single Reset per checkout; t.Setenv/t.Chdir modernization; host-git allowlist burn-down started with `internal/git/commit/commit_test.go` (TEST-6, TEST-8, TEST-5)
  - [ ] 02_help_docs_polish: root `promote` renamed/re-described (UX-2); deprecated `wt` alias resolution (UX-3); `camp projects` hint fix, Examples into Example fields, Long text for workitem/workflow trees (UX-10, UX-5); `camp status` real flags (UX-4); flow migrate help and history advertisement resolved (implement JSONL append or remove); cli-reference regenerated (BR-8); release-notes first-release cap (BR-9)
  - [ ] 03_misc_small_fixes: `--amend` no auto-stage plus `--no-edit` (GIT-6); index.lock age floor (GIT-9); marker-before-symlink (LINK-1); broken-symlink warning (LINK-2); dungeon `dungeon add` hint; pathsafe->pathutil merge (ARCH-8); shared slug package; ID-scheme matrix doc; UX-16/17/18 error-message batch
  - [ ] 04_upstream_dependency_tags: tag guild-scaffold and obey-shared, pin tags in go.mod (ARCH-9); disposition path if upstream access blocks
- **Verification:** `just gate` end to end; `just docs` produces no diff after regeneration; cli-reference includes `workitem priority`.

### Phase 004_POLISH (planning, verification-only per D006)

- Final verification checklist (created at scaffold time as the phase's working document):
  - [ ] `just gate` green end to end (both profiles)
  - [ ] `go vet -tags=integration ./...` clean
  - [ ] Zero `os.WriteFile` on R11/N-21 state paths (grep recorded)
  - [ ] No `.github/workflows/` changes in `git diff <branch-point>..HEAD --stat`
  - [ ] Every FESTIVAL_GOAL criterion checked with command output recorded
  - [ ] P2/P3 disposition review complete; data-loss/gate/boundary/contract classes fix-only
  - [ ] Regression/container test presence audit for every destructive-path fix
  - [ ] docs/json-contracts.md stability matrix current (JSON-6)

## Dependencies

| Item | Blocked by | Rationale |
|---|---|---|
| 02..12 | 01 | red gate must be green first; D007 records why this scope-capped bootstrap precedes the data-loss sequences |
| 04 | 02, 03 | FESTIVAL_GOAL order: data loss before gate enforcement (R13-R15 stay after data loss per D007) |
| 05..12 | 04 | dev-profile tests and integration vet must actually run before broad sweeps |
| 07.03 | 07.01, 07.02 | second-pass correction 3 |
| 10.05 | 09.08 | channel-aware templates need the Profile constant |
| 11.03 | 11.02 | R29 strictly before R30 |
| 11.05 | 03.04-03.06 | D003 |
| 004_POLISH | all of 003 | D006 verification-only |

## Risks and Mitigations

Carried from plan/STRUCTURE.md (file:line drift, N-4 blast radius, commitkit/fest compat, schema-bump coordination, red-gate-in-one-profile, dead-code over-deletion, upstream tag access). One addition: sequence 11.03's extraction is the largest mechanical diff in the festival; its task requires extraction-only commits (no behavior change) verified by the unchanged test suite, separate from any behavior fix.
