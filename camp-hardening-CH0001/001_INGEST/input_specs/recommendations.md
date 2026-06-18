# Recommendations: Prioritized Actionable List

Each item: what, why, where, suggested approach, severity, complexity in steps. Ordered for festival planning: P0 items are correctness/data-safety/launch blockers, P1 high-value hardening, P2 consistency and debt, P3 polish. IDs reference the findings docs.

---

## P0: Fix now (broken gates, data loss, boundary violations)

### R1. Green the test gate: manifest allowlist for `workitem priority` (TEST-1) | severity: critical | steps: 2
- **Where:** `cmd/camp/manifest_test.go` (~:150 count block, ~:209 allowlist block).
- **Approach:** add `"workitem priority": true` to the allowlist, bump the workitem restricted-command count; run unit suite in BOTH profiles to confirm green.

### R2. Stop `camp worktrees clean` from deleting uncommitted work (GIT-1) | severity: critical | steps: 4
- **Where:** `cmd/camp/worktrees/clean.go:228-251`, `internal/worktree/paths.go:116-133`.
- **Approach:** only fall through to `os.RemoveAll` for the "gitdir target does not exist" staleness reason; never auto-remove entries whose `.git` is a directory; require `--force` plus a dirty-tree check for anything else; add container tests (package currently has zero tests).

### R3. Add fest-ownership guards to dungeon triage (WF-1, WF-2) | severity: high | steps: 4
- **Where:** `internal/dungeon/service_list.go:125-136`, `internal/dungeon/resolver.go:64-72`.
- **Approach:** add `festivals` and `projects` to the built-in exclusion map; make the resolver skip any `dungeon/` directory whose parent chain includes `festivals/`; add regression tests for both; consider scaffolding a default `dungeon/.crawl.yaml` at init.

### R4. Make repair/workflow migrations all-or-nothing and never auto-commit failures (INIT-1, WF-3, WF-4) | severity: high | steps: 6
- **Where:** `internal/scaffold/repair.go:758-790`, `cmd/camp/init/command.go:262-284`, `internal/workflow/service_sync_migrate.go:184-187`.
- **Approach:** pre-validate every (src exists, dst absent) pair before moving anything (files clobber silently under POSIX rename); on any failure skip `commitRepairChanges`, exit non-zero with recovery guidance; reuse `statuspath.ExistingItemPath`; same destination check in MigrateV1ToV2 with per-item error collection.

### R5. Fix argv flattening in camp run just-dispatch and flow runner (RUN-1, flow runner) | severity: high | steps: 4
- **Where:** `cmd/camp/cmdutil/command.go:22-27`, `cmd/camp/run.go:161,183,192`, `internal/flow/runner.go:32-37`.
- **Approach:** project dispatch execs `just` directly with argv (`exec.CommandContext(ctx, "just", args...)`, `cmd.Dir = projectDir`); keep `sh -c` only for the documented raw-command form; flow runner passes extra args as positionals (`sh -c '<cmd> "$@"' sh args...`). Add tests with spaces/quotes/metacharacters in args.

### R6. Scope parent-ref sync commits (GIT-2, GIT-7) | severity: high | steps: 4
- **Where:** `cmd/camp/project/commit.go:196-207`, `pkg/commitkit/commitkit.go:186-213`, `cmd/camp/refs/commands.go:94-131`.
- **Approach:** commit with `CommitOptions.Only: []string{projectRelPath}` (support exists at `internal/git/commit.go:62-65`) in syncParentRef and SyncSubmoduleRef; refs-sync scopes with `Only: toSync` and prints skipped submodules. Coordinate the commitkit change with fest (it is fest's public API).

### R7. Stop `camp pull all` aborting rebases it did not start (GIT-3) | severity: high | steps: 3
- **Where:** `cmd/camp/pull.go:305-314`.
- **Approach:** probe `IsRebaseInProgress` before pulling each repo and skip with a warning; only abort a rebase the loop initiated; container test with a pre-existing conflicted rebase.

### R8. Fix `camp doctor --json` exiting 0 on failures (UX-11) | severity: high | steps: 2
- **Where:** `cmd/camp/doctor.go:114-121`.
- **Approach:** apply `exitDoctorWithCode(result)` after JSON output (or return a typed CommandError); add a test asserting non-zero exit with failing checks under --json.

### R9. Fix the README install command and the flow channel contradiction (BR-6, BR-7) | severity: high (launch) | steps: 4
- **Where:** `README.md:28`, `internal/scaffold/campaign/templates/.campaign/skills/campaign-workflows/SKILL.md.tmpl:33-35`, `templates/dungeon/OBEY.md.tmpl:36-42`, `dungeon add` Long help.
- **Approach:** correct the install path to `.../camp/cmd/camp@latest`; DECIDE flow's channel: either promote `flow` to stable or make scaffolded skills and help text channel-aware. Scaffolded skills instructing unknown commands is a first-run failure for every stable user.

### R10. Intent editor-path overwrite guard (WF-5) | severity: high | steps: 2
- **Where:** `internal/intent/service.go:199-206`, `internal/intent/helpers.go:46-49`.
- **Approach:** apply the same ErrFileExists guard CreateDirect uses, ideally `O_CREATE|O_EXCL`; test the edited-id collision case.

## P1: Hardening (concurrency, atomicity, dev-channel gates)

### R11. Atomic-write sweep (WF-6) | severity: high | steps: 6
- **Where:** intent (`internal/intent/service.go:120,292,354`, `rename.go:49,57`, `notes.go:111,162`), quest (`internal/quest/quest.go:271`), workflow (`service_init.go:58`, `service_sync_migrate.go:101,221`), flow (`internal/flow/registry.go:76`), state (`internal/state/location.go:105`), gitignore append (`internal/scaffold/repair.go:558-570`), private clone (`internal/commands/workitem/create.go:249-259`).
- **Approach:** route everything through `fsutil.WriteFileAtomically`; delete the private clone; make `LoadHistory` skip corrupt lines. Single highest-leverage durability fix.

### R12. Lock the registry and priority-store read-modify-write cycles (REG-1, workitem priority lock) | severity: high | steps: 5
- **Where:** `internal/config/registry.go:26-86` and mutating callers; `internal/commands/workitem/priority.go:85-97`; TUI `internal/workitem/tui/update.go:296-319`; pattern to copy: `internal/workitem/links/save.go:72-97`.
- **Approach:** `config.UpdateRegistry(ctx, mutate func(*Registry) error)` taking `fsutil.AcquireFileLock`, re-loading inside the lock; `priority.WithLock` mirroring links.WithLock; fix the fsutil stale-lock TOCTOU (`internal/fsutil/lock.go:71-86`) with rename-steal or PID liveness while there.

### R13. Run dev-tagged tests in a gate; add minimal CI (BR-1, BR-2) | severity: high | steps: 5
- **Where:** `internal/buildutil/tasks/test.go:81-84`, `.justfiles/test.just`, `.justfiles/release.just`, new `.github/workflows/`.
- **Approach:** make tasks.Test honor BUILD_TAGS; add `just test dev`; release recipes gate on tests in both profiles before tagging; minimal CI: build both profiles, vet both tag sets (including `-tags=integration`), unit tests both profiles, lint.

### R14. Delete or migrate the five dead integration-tagged host test files (TEST-2) | severity: high | steps: 4
- **Where:** `cmd/camp/{commit,pull,status}_integration_test.go`, `cmd/camp/refs/commands_integration_test.go`, `internal/git/lock_integration_test.go` (commit one does not compile: `commit_integration_test.go:317` undefined syncParentRef).
- **Approach:** migrate coverage worth keeping into `tests/integration/`, delete the rest; add `go vet -tags=integration ./...` to lint so tagged code cannot rot.

### R15. Fix container pool Reset for the .obey path move (TEST-3) | severity: high | steps: 2
- **Where:** `tests/integration/helpers_container.go:21-35`, `tests/integration/settings_test.go:16`.
- **Approach:** add `/root/.obey` and per-test work roots to the rm list; drop legacy `/root/.config/camp`; remove the manual workaround in settings_test.

### R16. Test internal/prune before it deletes more branches (TEST-4) | severity: high | steps: 4
- **Where:** `internal/prune/prune.go` (618 lines, zero tests; `deleteRemoteBranches` at :153 pushes remote deletions).
- **Approach:** table-driven unit tests of candidate/forced/merged classification with a fake executor; container test for `camp project prune` including `--remote-delete` against a bare origin.

### R17. Remove the `reg` alias collision and set SilenceUsage on root (UX-1, UX-12) | severity: high | steps: 2
- **Where:** `cmd/camp/register.go:36` vs `cmd/camp/registry/root.go:10`; `cmd/camp/root.go:44`.
- **Approach:** keep `reg` on registry only; `SilenceUsage: true` on root and remove the per-command patches; add a no-duplicate-aliases test.

### R18. Workitem list must stop writing during reads (workitem prune side effect) | severity: medium-high | steps: 3
- **Where:** `internal/commands/workitem/workitem.go:87-91`.
- **Approach:** move `priority.Prune` to explicit write paths (priority command, TUI mutation, `workitem doctor --fix`); this also stops transient discovery gaps permanently deleting priorities, which festival-app depends on.

## P2: Consistency and contracts

### R19. Standardize JSON flags and adopt the error envelope CLI-wide (UX-7, UX-8, UX-21, UX-20) | severity: medium | steps: 6
- **Approach:** `--json` everywhere (alias it on the `--format` commands); rename `flow add --json` input flag to `--from-json`; wrap remaining --json commands in `jsoncontract.RunE`; fix the status-all empty-case human string under --json.

### R20. Version the intent JSON contract (JSON-2) | severity: medium | steps: 3
- **Where:** `cmd/camp/intent/list.go:213-231`, `find.go:166-173`, `count.go:102-107`.
- **Approach:** versioned envelope mirroring the workitem payload; migrate to `--json` with the error contract; add the row plus `workitem priority` (JSON-1) to docs/json-contracts.md.

### R21. Version stamping and channel visibility (BR-3, BR-4) | severity: medium | steps: 4
- **Where:** `justfile:14` (dead ldflags var), `.justfiles/build.just:69-104`, `internal/version/version.go`.
- **Approach:** thread stamping into cross-platform recipes; `debug.ReadBuildInfo()` fallback; add a `Profile` field via build-tag constant pair, surfaced in `camp version --json` so festival-app can preflight dev-only commands.

### R22. Exit code table and os.Exit cleanup (UX-13, UX-14) | severity: medium | steps: 4
- **Approach:** define one CLI-wide table (0 ok, 1 runtime, 2 usage, 3+ documented partial states); align doctor/sync/clone options; replace os.Exit in sync/doctor/clone with typed CommandError.

### R23. Signal-aware root context (ERR-2, ERR-3) | severity: medium | steps: 3
- **Where:** `cmd/camp/main.go`, `cmd/camp/root.go:83`, `internal/git/retry.go:159`.
- **Approach:** `signal.NotifyContext` plus `ExecuteContext`; delete nil-ctx fallbacks; ctx-aware backoff sleep in retry.

### R24. fmt.Errorf lint ratchet (ERR-1) | severity: medium | steps: 4
- **Approach:** forbidigo rule banning fmt.Errorf outside tools/ with a burn-down allowlist (mirror the host-fs-test ratchet); migrate `internal/git/query.go` and `pkg/commitkit` first (widest propagation, and commitkit is fest's API).

### R25. Shared statusdir move primitive (workflow/dungeon/intent unification) | severity: medium | steps: 8
- **Where:** `internal/workflow/service_move.go`, `internal/dungeon/service_move.go`, `internal/intent/helpers.go`, `internal/scaffold/repair.go:767`, `internal/workflow/service_sync_migrate.go:184-187`.
- **Approach:** one package owning collision policy (no-replace semantics via O_EXCL link-then-remove or renameat2), dated buckets, cross-device fallback; migrate the four implementations; fixes the dungeon TOCTOU and stat-conflation findings as a side effect; then decide whether schema-driven workflow supersedes hardcoded dungeon.

### R26. Stage vocabulary and filter fixes feeding festival-app (workitem stage findings) | severity: medium | steps: 4
- **Where:** `internal/commands/workitem/workitem.go:230`, `internal/workitem/discover_workflow_docs.go:77`, `filter.go:21`, `discover_festivals.go:34`.
- **Approach:** LifecycleStage constants with per-type valid sets; represent "no stage" explicitly so --stage stops hiding design/explore items; publish per-type stage vocabulary in the JSON payload (schema bump); include or document ritual/ and chains/ festivals.

### R27. Shell wrapper transparency (SH-1, SH-2) and nav existence checks (NAV-1, NAV-2, NAV-3) | severity: medium | steps: 5
- **Approach:** drop `2>/dev/null` and check exit codes in the go/switch wrappers; add `--print` passthrough to go/switch branches; stat resolved paths before printing with one forced rebuild retry; reject `--list` with `--print`; lock the index rebuild.

### R28. Repair quiet failures and TTY guards (INIT-2, ERR-5, UX-19) | severity: medium | steps: 4
- **Approach:** registry-save failure during init returns the error; fest-init failure prints the cause; warn on swallowed cleanups; add IsTerminal guards plus agent_allowed annotations to intent explore, settings, intent crawl, dungeon crawl.

## P3: Debt and polish

### R29. Dead code sweep (ARCH-5) | severity: medium | steps: 5
- Delete `internal/clone` validation framework, `internal/concept.FSService`, `internal/nav` query/picker, `internal/shortcuts` Expander/FeedbackWriter, `internal/sync` preflights, plus the long tail; ~2,000+ lines. Do this BEFORE the cmd-logic extraction so nothing dead gets migrated.

### R30. Command-tree convention and cmd-logic extraction (ARCH-1, ARCH-2) | severity: medium | steps: 10
- Document Tree B constructor style as canonical; extract pull, status_all, shortcuts diff/reset, leverage scoring, dungeon crawl staging, go.go resolution into internal/; move EnsureRefForCommit into internal/workitem; consolidate run.go's duplicated resolution.

### R31. Git plumbing convergence and porcelain fixes (GIT-4, GIT-5, GIT-10) | severity: medium | steps: 6
- Fix the status_all TrimSpace porcelain miscount (use the `-z` parser at `internal/commands/workitem/staging_git.go:98-117`); add `-z` to FilterTracked; converge on RunGitCmd; literal exclusion pathspecs; `LC_ALL=C` on git execs; worktree-vs-submodule detection via gitdir target (GIT-8).

### R32. Quest robustness (quest findings) | severity: medium | steps: 4
- Skip-and-warn in List instead of hard-fail; atomic quest writes (in R11); Mkdir-claim slug reservation; decide the stable-build quest surface story (--quest flag and scaffolding visible in stable, BR-5).

### R33. Intent parser hardening | severity: medium | steps: 5
- Line-anchored frontmatter delimiters; round-trip unknown keys; O_EXCL on guarded writes; verify id in removeAllCopies; surface gather archive failures.

### R34. Flaky-test fixes and suite hygiene (TEST-6, TEST-8) | severity: low-medium | steps: 4
- Event-driven lock-release tests; os.Chtimes over sleeps; single Reset per checkout; t.Setenv/t.Chdir modernization; start burning down the 34-file host-git allowlist.

### R35. Help/docs polish batch (UX-2, UX-3, UX-5, UX-10, BR-8, flow help contradictions) | severity: low | steps: 5
- Rename or re-describe root `promote`; fix the `camp projects` hint; regenerate cli-reference (workitem priority missing); Examples into the Example field; Long text for the workitem/workflow trees; fix `flow migrate` help text and the unimplemented history advertisement (implement or remove).

### R36. Misc small fixes | severity: low | steps: 4
- `camp commit --amend` no auto-stage plus `--no-edit` (GIT-6); marker-before-symlink ordering in project link (LINK-1); broken-symlink warning in project list (LINK-2); dungeon hint text `dungeon init` to `dungeon add`; index.lock age floor (GIT-9); merge pathsafe into pathutil (ARCH-8); tag guild-scaffold and obey-shared (ARCH-9); shared slug package; document the ID-scheme matrix.

---

## Suggested festival shape

- **Phase 1 (stop the bleeding):** R1-R10. Independent, small, each verifiable.
- **Phase 2 (durability and gates):** R11-R18. R11/R12 first; R13-R15 unlock trustworthy gates for everything after.
- **Phase 3 (contracts and consistency):** R19-R28. R20/R21/R26 coordinate with festival-app; R25 is the big unification.
- **Phase 4 (debt):** R29-R36. R29 strictly before R30.
