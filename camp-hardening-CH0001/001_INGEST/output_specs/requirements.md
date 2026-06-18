# Requirements: camp-hardening (CH0001)

Every requirement is traceable to a finding ID in `input_specs/` (R# = recommendations.md, N-# = second-pass-fable.md, other IDs = the findings docs). Priorities follow the review's tiers as re-ordered by the second pass (section 4): the second pass promoted N-1, N-2, N-3, N-6 into P0 and required N-4/N-5 to land before or with R6.

## P0: Data loss, broken gates, boundary violations

### REQ-01. Worktrees clean must never destroy uncommitted work (R2; GIT-1; second-pass correction 1)
- `camp worktrees clean` falls through to `os.RemoveAll` ONLY for the "gitdir target does not exist" staleness reason.
- Entries whose `.git` is a directory (full clone or data dir parked under `projects/worktrees/`) are never auto-removed.
- A confirmation prompt exists before deletion (today the command deletes immediately after printing the list; only `--dry-run` saves you).
- The `projectPath == ""` branch (project not in campaign config) no longer bypasses git straight to `os.RemoveAll`.
- Anything else requires `--force` plus a dirty-tree check.
- Acceptance: container integration tests for the dirty-tree, `.git`-directory, empty-projectPath, and stale cases (package currently has zero tests).

### REQ-02. `camp cp --force` same-file and self-recursion guards (N-1)
- Reject `os.SameFile(src, dest)` before any open-for-write (today `O_TRUNC` zeroes the shared inode).
- Reject directory copies where dest is under src (today `CopyDir` recurses into its own output).
- Acceptance: container tests for both cases; mirror cp(1) error wording.

### REQ-03. fresh/prune dirty-worktree protection plus prune test coverage (N-2, R16; TEST-4)
- Branch-worktree removal in `internal/prune` runs the existing `detachedWorktreeClean`-style check and skips dirty trees with `SkipReasonDirtyWorktree`.
- An explicit flag is required to destroy dirty worktrees; `camp fresh` no longer hard-codes `Force: true` past a dirty check.
- Acceptance: table-driven unit tests of candidate/forced/merged classification with a fake executor; container test for `camp project prune` including `--remote-delete` against a bare origin (`deleteRemoteBranches` pushes remote deletions and has zero tests).

### REQ-04. `camp project remove` must not destroy unpushed history (N-3)
- Before removing `.git/modules/projects/<name>`, check for refs not reachable from any remote; refuse without `--force`, naming the branches that would be lost.
- Fix the help text claim that "project files remain in place" (false under `git rm -f`).
- Acceptance: container test with a committed-but-unpushed branch.

### REQ-05. `camp sync` stale-ref fallback safety (N-6, N-10)
- Never `os.RemoveAll` a non-empty submodule dir without explicit consent (quarantine-rename instead).
- Perform the fallback through real submodule machinery preserving `.git/modules` wiring; resolve relative `.gitmodules` URLs against the superproject remote, not process cwd.
- Surface the discarded `CheckoutDefaultBranch` error; correct `validateUpdate`'s backwards drift-skip logic; warn or fast-forward when branch tip != recorded gitlink so later ref syncs cannot silently regress pointers.
- Acceptance: container test with a hostile-remote fixture (force-pushed upstream), shared with REQ-08's fixture work.

### REQ-06. All-or-nothing migrations, no auto-commit of failures (R4: INIT-1, WF-3, WF-4)
- Repair `ExecuteMigrations` and workflow `MigrateV1ToV2` pre-validate every (src exists, dst absent) pair before any rename (POSIX rename clobbers files silently); reuse `statuspath.ExistingItemPath`.
- On any failure: exit non-zero with recovery guidance; `commitRepairChanges` never runs after a failed migration (today the error is downgraded to a warning and the half-migrated tree is committed as a successful repair).
- MigrateV1ToV2 collects per-item errors and applies the same destination check `service_move.go` already uses.
- Acceptance: unit tests for the pre-validation and failure paths; a repair-migration failure leaves git uncommitted.

### REQ-07. Intent moveFile overwrite guard (R10, N-19; WF-5) and promote rollback (N-12)
- The ErrFileExists guard lands in or around `moveFile` itself so both `CreateWithEditor` and `EditWithEditor` call sites are covered; ideally `O_CREATE|O_EXCL`.
- The copy fallback fires only on EXDEV (`errors.Is(err, syscall.EXDEV)`), never on arbitrary rename errors (today it opens dest with `os.Create`/O_TRUNC on ANY rename error).
- Intent promote rollback tracks directory creation with `os.Mkdir` (not MkdirAll-and-assume) and removes only what it created; a pre-existing `workflow/design/<slug>/` with user files survives a failed promote.
- Acceptance: edited-id collision test; rollback test with a pre-existing design dir.

### REQ-08. `camp pull all` rebase safety (R7; GIT-3)
- Probe `IsRebaseInProgress` before pulling each repo and skip with a warning; only abort a rebase the loop itself initiated.
- Acceptance: container test with a pre-existing conflicted rebase (fixture shared with REQ-05).

### REQ-09. Green the test gate (R1; TEST-1)
- Add `workitem priority` to the manifest allowlist, bump the restricted-command count; suite green in BOTH profiles.

### REQ-10. Fest-boundary guards in dungeon (R3: WF-1, WF-2)
- `festivals` and `projects` added to the built-in exclusion map in `service_list.go`.
- The resolver skips any `dungeon/` directory whose parent chain includes `festivals/`.
- Consider scaffolding a default `dungeon/.crawl.yaml` at init.
- Acceptance: regression tests for both the exclusion map and the resolver.

### REQ-11. `camp doctor --json` exit code (R8; UX-11)
- Apply `exitDoctorWithCode(result)` (or a typed CommandError) after JSON output.
- Acceptance: test asserting non-zero exit with failing checks under `--json`.

### REQ-12. Launch docs and first-run correctness (R9: BR-6, BR-7; R17: UX-1, UX-12)
- README install path corrected to `.../camp/cmd/camp@latest`.
- The flow channel contradiction DECIDED and fixed: either promote `flow` to stable or make scaffolded skills, templates, README, and `dungeon add` help channel-aware (scaffolded skills instructing unknown commands is a first-run failure for every stable user).
- `reg` alias kept on `registry` only; `SilenceUsage: true` on root with per-command patches removed; no-duplicate-aliases test added.

### REQ-18. Argv integrity in run/flow dispatch (R5: RUN-1)
- Project just-dispatch execs `just` directly with argv and `cmd.Dir = projectDir`; `sh -c` kept only for the documented raw-command form.
- Flow runner passes extra args as positionals (`sh -c '<cmd> "$@"' sh args...`).
- Acceptance: tests with spaces, quotes, and metacharacters in args.

### REQ-19. Scoped-commit semantics, in dependency order (N-4, N-5, then R6: GIT-2, GIT-7, then N-14)
- N-4 first: the `--only` engine commits index content not worktree content (temporary `GIT_INDEX_FILE` via `read-tree HEAD` plus `add -- <paths>`), or verifies index==worktree before `--only` and fails loudly on divergence.
- N-5: `listStagedFiles` parses `diff --cached --name-status -z` including both rename/copy sides.
- R6: `syncParentRef` and commitkit `SyncSubmoduleRef` commit with `Only: []string{projectRelPath}`; refs-sync scopes with `Only: toSync` and prints skipped submodules.
- N-14: root `camp commit` final commit scoped to the paths camp staged, or staged `projects/*` gitlinks detected pre-commit and refused without `--include-refs`.
- The fest-side mirror of the commitkit scoping (fest's own `commit.go:541-553` reimplementation) is captured as a fest workitem via `camp workitem create`; it is NOT fixed in this festival (second-pass correction 2).
- Note: R6 is P0 in the source corpus; N-4/N-5 are its mandatory prerequisites per second-pass correction 3, so the whole chain sits in P0.

## P1: Durability, concurrency, gates

### REQ-13. Atomic-write sweep (R11: WF-6, N-21)
- Route through `fsutil.WriteFileAtomically`: intent (`service.go:120,292,354`, `rename.go:49,57`, `notes.go:111,162`), quest (`quest.go:271`), workflow (`service_init.go:58`, `service_sync_migrate.go:101,221`), flow registry (`registry.go:76`), state (`location.go:105`), gitignore append (`repair.go:558-570`), workitem private clone (`create.go:249-259`, delete the clone), config allowlist (`allowlist.go:84`).
- `LoadHistory` skips corrupt lines instead of hard-failing.
- Acceptance: grep gate showing zero `os.WriteFile` on the listed paths.

### REQ-14. Registry and priority-store locking plus lock TOCTOU (R12: REG-1, REG-2)
- `config.UpdateRegistry(ctx, mutate func(*Registry) error)` taking `fsutil.AcquireFileLock` and re-loading inside the lock; migrate all mutating callers.
- `priority.WithLock` mirroring `links.WithLock`; TUI path included.
- Fix the fsutil stale-lock TOCTOU (rename-steal or PID liveness).

### REQ-15. Local gate enforcement, both profiles (R13: BR-1, BR-2; gate framing per second-pass section 4.7)
- `tasks.Test` honors `BUILD_TAGS`; `just test dev` (or equivalent) exists; release recipes gate on tests in both profiles before tagging.
- A pre-push hook runs build, test, and lint locally.
- HARD CONSTRAINT: no GitHub Actions workflows are added or modified; all enforcement is just recipes and hooks.

### REQ-16. Dead integration-tagged tests and vet coverage (R14: TEST-2)
- Migrate coverage worth keeping from the five dead files into `tests/integration/`; delete the rest (the commit one does not compile).
- `go vet -tags=integration ./...` added to `just lint`.

### REQ-17. Container pool Reset paths (R15: TEST-3)
- Add `/root/.obey` and per-test work roots to the Reset rm list; drop legacy `/root/.config/camp`; remove the manual workaround in `settings_test.go`.

### REQ-20. Reads do not write (R18, N-8, N-18)
- Workitem list prune side effect moved to explicit write paths (priority command, TUI mutation, `workitem doctor --fix`).
- Intent read commands stop calling the EnsureDirectories migration; migration moves to `camp init --repair` and explicit write paths; reads at most warn.
- `status all` stops writing the cache on read; the dead `--no-cache` flag is either removed or made real with an actual read path.

## P2: Contracts and consistency (festival-app coordination)

### REQ-21. flow items JSON contract (N-7)
- `flow items --json` emits a versioned payload via `jsoncontract`; errors to stderr with non-zero exit; `cmd.Context()` not `context.Background()`; remove or implement the dead `HistoryOptions.JSON`.

### REQ-22. workitem create atomicity (N-9)
- Derive id and ref before creating the target directory, or remove the directory on any post-MkdirAll failure path, so retry is never blocked by "use adopt" (FA0011 retry-model compatibility).

### REQ-23. Intent JSON contract versioning (R20: JSON-2, JSON-1)
- Versioned envelope for intent list/find/count/show mirroring the workitem payload; migrate to `--json` with the error contract; add the intent row plus `workitem priority` to `docs/json-contracts.md`.

### REQ-24. Stage vocabulary and filters (R26)
- LifecycleStage constants with per-type valid sets; explicit "no stage" representation so `--stage` stops hiding design/explore items; per-type stage vocabulary published in the JSON payload (schema bump); ritual/ and chains/ festivals included or documented.

### REQ-25. Intent type filter (N-15)
- `intent list --type` honors all provided values (client-side multi-type filter like statuses) or rejects more than one.

### REQ-26. Path semantics unification (N-16)
- Campaign-relative paths plus one `campaign_root` field across workitem, intent, quest, and status-all JSON surfaces.

### REQ-27. Manifest completeness (N-11)
- Every shipped command carries an `agent_allowed` annotation; doctor's read (`--json`) vs `--fix` story split.

### REQ-28. Version stamping and profile visibility (R21: BR-3, BR-4; N-25)
- ldflags stamping threaded into cross-platform recipes; `debug.ReadBuildInfo()` fallback; `Profile` field via build-tag constant pair surfaced in `camp version --json`; `--short` no longer silently beats `--json`; JSON marshal failure exits non-zero (convert version to RunE).

### REQ-29. JSON flag and envelope standardization (R19: UX-7, UX-8, UX-20, UX-21)
- `--json` everywhere (aliased on `--format` commands); `flow add --json` input flag renamed `--from-json`; remaining `--json` commands wrapped in `jsoncontract.RunE`; status-all empty case emits `{"timestamp":...,"repos":[]}` under `--json`.

### REQ-30. Remaining P2 batch (R22, R23, R27, R28; N-13, N-17, N-24)
- R22: one CLI-wide exit-code table; os.Exit in sync/doctor/clone replaced with typed CommandError.
- R23: `signal.NotifyContext` plus `ExecuteContext`; nil-ctx fallbacks deleted; ctx-aware backoff sleep in git retry.
- R27: shell wrappers drop `2>/dev/null` and check exit codes; `--print` passthrough on go/switch; nav stats resolved paths with one forced rebuild retry; `--list` with `--print` rejected; index rebuild locked.
- R28: init registry-save failure returns the error; fest-init failure prints the cause; swallowed cleanups warn; IsTerminal guards plus `agent_allowed` annotations on intent explore, settings, intent crawl, dungeon crawl.
- N-13: `camp pull`/`push` intercept only `--project`/`--project=`, honor `--`, fix the `-p <display-name>` hint.
- N-17: `camp concepts` standard error outside a campaign plus a versioned `--json`.
- N-24: `intent add` `-f` collision resolved; machine-readable created id/path output.

## P3: Debt and polish

### REQ-31. fmt.Errorf ratchet (R24: ERR-1)
- forbidigo rule banning fmt.Errorf outside tools/ with a burn-down allowlist; `internal/git/query.go` and `pkg/commitkit` migrated first.

### REQ-32. Shared statusdir move primitive (R25)
- One package owning collision policy (no-replace semantics), dated buckets, cross-device fallback; the four implementations (workflow, dungeon, intent, repair/sync-migrate) migrated; fixes the dungeon TOCTOU and stat-conflation findings as a side effect.

### REQ-33. Dead code sweep strictly before cmd extraction (R29 then R30: ARCH-5, ARCH-1, ARCH-2)
- R29: delete `internal/clone` validation framework, `internal/concept.FSService`, `internal/nav` query/picker, `internal/shortcuts` Expander/FeedbackWriter, `internal/sync` preflights, plus the long tail (~2,000+ lines).
- R30 (after R29): Tree B constructor style documented canonical; pull, status_all, shortcuts diff/reset, leverage scoring, dungeon crawl staging, go.go resolution extracted into internal/; `EnsureRefForCommit` into `internal/workitem`; run.go duplicated resolution consolidated.

### REQ-34. Git plumbing convergence (R31: GIT-4, GIT-5, GIT-8, GIT-10)
- status_all uses the `-z` porcelain parser; `FilterTracked` gets `-z`; converge on RunGitCmd; literal exclusion pathspecs; `LC_ALL=C` on git execs; worktree-vs-submodule detection via gitdir target.

### REQ-35. Quest robustness (R32)
- Skip-and-warn in List; atomic quest writes (in REQ-13); Mkdir-claim slug reservation; stable-build quest surface story decided (BR-5).

### REQ-36. Intent parser hardening (R33; plus N-26)
- Line-anchored frontmatter delimiters; unknown keys round-tripped; O_EXCL on guarded writes; id verified in removeAllCopies; gather archive failures surfaced; gather's orphaned half-written intent on metadata Save failure cleaned up.

### REQ-37. Flaky-test hygiene (R34: TEST-6, TEST-8)
- Event-driven lock-release tests; os.Chtimes over sleeps; single Reset per checkout; t.Setenv/t.Chdir modernization; host-git allowlist burn-down started.

### REQ-38. Help and docs polish (R35: UX-2, UX-3, UX-5, UX-10, BR-8)
- Root `promote` renamed or re-described; `camp projects` hint fixed; cli-reference regenerated (workitem priority missing); Examples in the Example field; Long text for workitem/workflow trees; `flow migrate` help fixed; unimplemented history advertisement implemented or removed.

### REQ-39. Misc small fixes (R36; N-20, N-22, N-23, N-27)
- `--amend` no auto-stage plus `--no-edit` (GIT-6); marker-before-symlink ordering (LINK-1); broken-symlink warning in project list (LINK-2); dungeon hint text fix; index.lock age floor (GIT-9); pathsafe merged into pathutil (ARCH-8); guild-scaffold and obey-shared tagged (ARCH-9); shared slug package; ID-scheme matrix documented; N-20 (skills `--force` unmanaged-symlink protection); N-22 (pins migration symlinked-root fix); N-23 ("no changes added to commit" classified as ErrNoChanges); N-27 (buildutil clean cwd rm-rf flagged/guarded).

## Disposition rule

All P0/P1 requirements must be FIXED with regression tests. P2/P3 items must be fixed or explicitly dispositioned in the festival decision log with a reason; "dispositioned" is not available for anything in the data-loss, gate, boundary, or festival-app-contract classes.
