# camp Second-Pass Adversarial Review

**Date:** 2026-06-11
**Repo state:** `projects/camp` at `11c270f` (two commits past the first-pass baseline `5c22d2b`; the new commits are intent TUI work and do not touch any finding site). Working tree carries uncommitted intent/tui modifications, untouched by this review.
**Nature:** read-only verification plus fresh hunt. No camp files were modified. This file is the only write.
**Method:** every critical and high first-pass finding (plus the festival-app-gating mediums) was re-read at the cited file:line with a default posture of refute. The red gate and the dead integration tests were re-run live in both profiles. The fresh hunt ran as three parallel deep-dive passes (data-loss, JSON contract surface, git/concurrency); every load-bearing sub-pass claim was then independently re-verified by direct read before inclusion, and two of the git-semantics claims were verified by the sub-pass empirically in scratch repos outside this repo.

---

## 1. Verification table

Verdicts: CONFIRMED (code matches the claim at the cited or corrected lines), REFUTED, ADJUSTED. Where the second pass found the situation WORSE than reported, that is noted inline; severity was never found overstated.

| # | Finding | First-pass severity | Verdict | Evidence |
|---|---|---|---|---|
| 1 | TEST-1 red unit test gate on main | critical | CONFIRMED, still red at `11c270f` | Ran live this session. Stable: `expected exactly 27 restricted commands, got 28` and `"workitem priority" is agent_allowed=true but not in allowlist` (`cmd/camp/manifest_test.go:162,222`). Dev (`-tags dev`): same two failures at 43 vs 44. |
| 2 | TEST-2 dead integration-tagged tests, one non-compiling | critical | CONFIRMED | Ran `go vet -tags=integration ./cmd/camp/...` live: fails with `cmd/camp/commit_integration_test.go:317:12: undefined: syncParentRef` (the function exists only in package `cmd/camp/project` now). |
| 3 | GIT-1 worktrees clean RemoveAll data loss | critical | CONFIRMED, scope WORSE | `cmd/camp/worktrees/clean.go:241-247`: `git.Remove` error swallowed (`// Fall through to filesystem removal`), then unconditional `os.RemoveAll`. Staleness misclassification confirmed at `:194-202` (`missing .git file`, `.git is a directory (not a worktree)`). `internal/worktree/paths.go:116-133` enumerates every non-dot directory. Two aggravations the first pass did not state: (a) there is NO confirmation prompt at all; `runWorktreesClean` prints the list then deletes immediately (`clean.go:106-138`), only `--dry-run` saves you; (b) when the project name is not in campaign config, `projectPath` stays empty and `git.Remove` is skipped entirely, going straight to `os.RemoveAll` (`clean.go:230-247`). |
| 4 | WF-1 dungeon triage offers `festivals/` | high | CONFIRMED | Exclusion map at `internal/dungeon/service_list.go:125-136` verbatim: no `festivals`, no `projects`. Mitigation absence re-verified in the live campaign root: `festivals/dungeon` exists; no root `.workflow.yaml`, no `dungeon/.crawl.yaml`, no `.crawlignore`, no `festivals/OBEY.md`. |
| 5 | WF-2 dungeon resolver adopts `festivals/dungeon/` | high | CONFIRMED | `internal/dungeon/resolver.go:64-75`: first `<dir>/dungeon` walking up from cwd wins, no fest-ownership check. |
| 6 | WF-3 repair migration rename clobber | high | CONFIRMED | `internal/scaffold/repair.go:767`: `os.Rename(src, dst)` with no dst stat, first error aborts mid-sequence (`:768`). |
| 7 | INIT-1 partial migration auto-committed | high | CONFIRMED | `cmd/camp/init/command.go:263-267`: error downgraded to a warning line; `:282-284` `commitRepairChanges` runs unconditionally for `p.Repair && !p.DryRun`. |
| 8 | WF-4 MigrateV1ToV2 rename onto root, no collision check | high | CONFIRMED | `internal/workflow/service_sync_migrate.go:181-187`: `dst := filepath.Join(s.root, name)` then bare `os.Rename`; first error returns with schema still v1; v2 schema write at `:221` is plain `os.WriteFile`. |
| 9 | RUN-1 cr/camp run sh -c argv flattening | high | CONFIRMED | `cmd/camp/cmdutil/command.go:22-27`: `strings.Join(extraArgs, " ")` into `sh -c`. Project just-dispatch routes through it with args at `cmd/camp/run.go:183`; `run.go:161,192` also flatten with `strings.Join`. |
| 10 | GIT-2 syncParentRef and commitkit SyncSubmoduleRef sweep pre-staged changes | high | CONFIRMED, blast radius slightly different | `cmd/camp/project/commit.go:196-207`: stages relPath, commits unscoped. `pkg/commitkit/commitkit.go:186-213`: stages only the ref (`:192`) but `HasStagedChanges` is whole-repo (`:197`) and `git.Commit` unscoped (`:208`). Fest-facing claim verified with a nuance: fest's grep shows no direct `SyncSubmoduleRef` call, but `projects/fest/internal/commands/commit/commit.go:541-553` independently reimplements the identical unscoped pattern from commitkit primitives (StageFiles, whole-repo HasStagedChanges, unscoped Commit), so the wrong-content-commit exposure through fest is real either way. |
| 11 | GIT-3 pull all aborts pre-existing rebases | high | CONFIRMED | `cmd/camp/pull.go:306-314`: `handlePullError` checks `IsRebaseInProgress` only after a failed pull and `_ = abortRebase` with no did-we-start-it tracking. |
| 12 | UX-11 doctor --json exits 0 on failures | high | CONFIRMED | `cmd/camp/doctor.go:114-121`: JSON branch returns `outputDoctorJSON(result)`; `exitDoctorWithCode` reached only in text mode. |
| 13 | REG-1 registry load-modify-save unlocked | high | CONFIRMED | `internal/config/registry.go`: only `fsutil` reference is `WriteFileAtomically` at `:81`; zero `AcquireFileLock` usage in the file. |
| 14 | Workitem priority store race (no lock) | medium | CONFIRMED | `internal/commands/workitem/priority.go:85-98`: `priority.Load` then `priority.SaveOrDelete` with no file lock, unlike `links.WithLock`. |
| 15 | Workitem list mutates on read (prune side effect) | medium | CONFIRMED | `internal/commands/workitem/workitem.go:77-91`: `priority.Prune` plus `SaveOrDelete` inside the list RunE, including the `--json` path. |
| 16 | Stage filter hides design/explore items | medium | CONFIRMED | `internal/workitem/discover_workflow_docs.go` sets `LifecycleStage: ""` for workflow-doc items; `internal/workitem/filter.go:21-22` `stageSet[item.LifecycleStage]` can never match the empty stage for any normal `--stage` value. |
| 17 | WF-5 CreateWithEditor silent intent overwrite | high | CONFIRMED, scope WORSE | `internal/intent/service.go:198-206`: no stat guard before `moveFile`; `internal/intent/helpers.go:46-50` `moveFile` starts with bare `os.Rename` and its fallback opens dest with `os.Create` (O_TRUNC) on ANY rename error, not just EXDEV. Second pass found the same clobber on a second call site: `EditWithEditor`'s status-change path at `service.go:259` (see new finding N-19). |
| 18 | Quest List hard-fail bricks quest commands | medium | CONFIRMED | `internal/quest/quest.go:134-137`: `return nil, err` on the first `Load` failure; quest write at `:271` is plain `os.WriteFile` (non-atomic, as listed in WF-6). |
| 19 | UX-20 status all --json human string in the empty case | medium | CONFIRMED | `cmd/camp/status_all.go:104-107`: `fmt.Println(ui.Info(...))` and `return nil` before the `statusAllJSON` branch at `:120`. |
| 20 | JSON-2 intent JSON unversioned ad hoc contract | medium | CONFIRMED | `cmd/camp/intent/list.go:212-231`: inline `jsonIntent` struct, bare top-level array, no `schema_version`, triggered via `--format json` so no error envelope. |

**Tally: 20 verified, 20 CONFIRMED, 0 REFUTED, 0 ADJUSTED.** Two findings (GIT-1, WF-5) are worse than first-pass scope; one (GIT-2) has a corrected fest blast-radius mechanism with the same conclusion.

---

## 2. Corrections

Nothing was refuted and no severity was overstated. Three precision corrections to carry into remediation planning:

1. **GIT-1 (R2):** the fix must also cover (a) the missing confirmation step (today the command deletes immediately after printing the list) and (b) the `projectPath == ""` branch that bypasses git entirely. The first-pass framing "ignoring --force" is accurate but understated: `--force` only changes the swallowed `git worktree remove` attempt; removal happens identically with or without it.
2. **GIT-2 (R6):** fest's exposure comes through its own `commit.go:541-553` reuse of commitkit primitives in the same unscoped pattern, not (only) through `SyncSubmoduleRef`. Fixing `SyncSubmoduleRef` alone does not close the fest side; the fest repo needs the same scoping change, so R6's "coordinate with fest" step is mandatory, not optional.
3. **R6/R5 implementation caveat (from new finding N-4):** the recommended fix "commit with `CommitOptions.Only`" relies on `git commit --only`, which commits the WORKTREE content of the listed paths, not the staged blobs. For submodule gitlinks this is acceptable (worktree gitlink state is the submodule HEAD), but anyone extending the `Only` pattern to file paths inherits the over-commit semantics described in N-4. The remediation festival should fix N-4 before or together with R6, not after.

---

## 3. New findings (fresh hunt)

Everything below was absent from the first-pass docs. Each load-bearing claim was re-verified by direct read in this session; N-4 and N-5 were additionally reproduced empirically in scratch repos by the git deep-dive pass.

### Critical

**N-1. `camp cp --force` zeroes source files when src and dest resolve to the same path; copying a directory into itself recurses.**
- Where: `cmd/camp/copy.go:75-101`, `internal/transfer/copy.go:12-34`.
- Evidence: when dest is a directory, `dest = filepath.Join(dest, filepath.Base(src))` (`copy.go:76-77`). There is no `os.SameFile` or dest-under-src guard anywhere. `transfer.CopyFile` opens dest with `os.OpenFile(dest, O_CREATE|O_WRONLY|O_TRUNC, ...)` (`copy.go:24` in transfer), truncating the shared inode to zero before the read. `camp cp -f notes.md .` from the file's own directory empties the file; `camp cp -f mydir .` empties every file in the tree via `CopyDir`. The non-force error message ("use --force to overwrite") actively steers the user into the destructive invocation. `CopyDir` also `MkdirAll`s dest before walking src, so a dest inside src recurses into its own output.
- Why it matters: one habitual cp-style invocation destroys user content irrecoverably, in a command marketed for moving campaign documents.
- Fix: stat both sides and reject `os.SameFile`; reject dest paths under src for directories; mirror cp(1)'s "same file" and "cannot copy a directory into itself" guards. Add container tests for both cases.

**N-2. `camp fresh` (and `camp project prune`) force-removes DIRTY worktrees of merged branches with no dirty check and, under fresh, no prompt.**
- Where: `internal/prune/prune.go:472-482` (`wt.Remove(ctx, entry.Path, true)`, force hardcoded true), `internal/commands/fresh/fresh.go:217-227` (`Force: true, // Skip confirmation — fresh is deliberate`).
- Evidence: the branch-worktree removal path has no clean check. The detached-worktree path right next to it DOES check (`detachedWorktreeClean` at `prune.go:302-323`, skipping with `SkipReasonDirtyWorktree`), which marks the branch path as an oversight rather than a decision. `camp fresh` passes `Force: true` so the interactive prompt is skipped entirely; even the interactive `camp project prune` prompt does not check or mention dirtiness.
- Why it matters: keep a worktree for a merged PR with uncommitted follow-up edits, run routine `camp fresh`, and the worktree is `git worktree remove --force`d, uncommitted and untracked work gone. `camp fresh all` repeats this across every project. Combined with TEST-4 (prune has zero tests), this is the most dangerous untested code in the repo.
- Fix: run the existing `detachedWorktreeClean`-style check on branch worktrees and skip dirty ones with `SkipReasonDirtyWorktree`; require an explicit flag to destroy dirty worktrees. Fold into R16's test work.

### High

**N-3. `camp project remove` always deletes `.git/modules/projects/<name>`, destroying unpushed branches and commits, even without `--delete`.**
- Where: `internal/project/remove.go:167-176`.
- Evidence: after `deinit -f` plus `git rm -f`, the code unconditionally `os.RemoveAll`s the module gitdir ("Always clean up .git/modules after a successful submodule removal, regardless of whether --delete was requested"). The only guard is `isDirtySubmodule` (`:147-158`), which is `git status --porcelain` and passes when work is committed. Git deliberately preserves `.git/modules` on deinit/rm precisely so local history survives; committed-but-unpushed branches live only there. The command help also claims "project files remain in place for you to handle manually", which `git rm -f` makes false.
- Why it matters: a user unregistering a project with clean status but unpushed local branches permanently loses that history with no warning and no `--force` required.
- Fix: before removing the module dir, check for refs not reachable from any remote (`for-each-ref` plus cherry/upstream comparison); refuse without `--force` and name the branches that would be lost.

**N-4. The scoped auto-commit engine commits WORKTREE content, not the index: `git commit --only` semantics.**
- Where: `internal/git/commit/commit.go:83-107` (`stageAndCommit` builds `Only: expandedScope`), `internal/git/commit.go:62-65` (`--only --` argv).
- Evidence (reproduced in a scratch repo by the git pass; argv path re-verified by read): with `f.txt` staged at v2 and worktree at v3, `git commit --only -- f.txt` commits v3 and silently restages it. Consequences: (a) `camp workitem commit --staged` (`internal/commands/workitem/commit.go`, plan from `staging.go`) documents "commit whatever is already in the git index" but actually folds in unstaged worktree edits to those same files; (b) every quest/intent/workitem/dungeon/repair auto-commit has a window between staging/expansion and commit in which a concurrent camp or fest process writing an in-scope file gets its half-written content captured under this commit's tag.
- Why it matters: the scoped-commit engine is the first pass's praised centerpiece and the foundation R6 builds on; its core primitive does not have the snapshot semantics everyone, including the in-repo docs, assumes.
- Fix: commit from the index (temporary `GIT_INDEX_FILE` built via `read-tree HEAD` plus `add -- <paths>`), or at minimum verify `git diff --quiet -- <paths>` (index == worktree) before using `--only` and fail loudly on divergence.

**N-5. `camp workitem commit --staged` corrupts staged renames: the delete side is dropped and a stray staged `D` is left behind.**
- Where: `internal/commands/workitem/staging_git.go:55-61` (`listStagedFiles` uses `diff --cached --name-only -z`), interacting with `internal/git/commit.go:255` (`ExpandTrackedPaths` re-scoped to dest-only pathspec).
- Evidence (reproduced by the git pass; plumbing re-verified by read): a staged `git mv a.txt b.txt` yields only `b.txt` from `--name-only`; the dest-scoped `--name-status` diff degrades the rename pair to `A b.txt`; `git commit --only -- b.txt` commits the add while `a.txt` stays in HEAD and `D a.txt` stays staged. The leftover staged deletion then either blocks subsequent workitem commits (`assertCleanIndex`) or gets swept into a later unrelated `--staged` commit under the wrong message.
- Fix: `listStagedFiles` should parse `diff --cached --name-status -z` and include both rename/copy sides, using the same parser `ExpandTrackedPaths` already has.

**N-6. `camp sync` stale-ref fallback `os.RemoveAll`s the submodule directory unconditionally, then replaces it with a NON-submodule clone at the remote default branch.**
- Where: `internal/git/submodule_init.go:158-172` (`InitFromDefaultBranch`), reached from `internal/sync/sync.go:234` via `InitSubmoduleGraceful`.
- Evidence (re-verified by read; found independently by both the data-loss and git passes): the comment says "Remove empty submodule directory if exists" but `os.RemoveAll(subDir)` runs with no emptiness or dirtiness check, and under `camp sync --force` dirty/untracked content is warnings-only, so uncommitted work is silently destroyed. The replacement is a plain `git clone url subDir` with no `cmd.Dir`: it creates a standalone `.git` DIRECTORY (the path stops being a submodule, orphaning `.git/modules/<name>`), checks out the remote default branch instead of the recorded SHA, and a relative `.gitmodules` URL (`../foo.git`) resolves against the camp process cwd rather than the superproject remote. The per-submodule result still reports success.
- Why it matters: a force-push upstream or a teammate's unpushed pointer triggers this path during a routine `camp sync --force`; user data is deleted, and the repo is left structurally wrong (worktree-vs-submodule misdetection GIT-8 then compounds).
- Fix: refuse to RemoveAll a non-empty dir without explicit consent (quarantine-rename instead); perform the fallback through real submodule machinery (`git -C <gitdir> fetch` then `submodule update --init`) preserving the `.git/modules` wiring; resolve relative URLs against the superproject remote.

**N-7. `camp flow items --json` never outputs JSON; the flag is dead, errors go to stdout, exit code lies.**
- Where: `internal/commands/flow/items.go:33,60-76`; `internal/workflow/service.go:177`.
- Evidence (re-verified by read): the flag feeds `workflow.ListOptions{JSON: flowItemsJSON}` but nothing reads `ListOptions.JSON`; the RunE always prints the human table. Errors are `fmt.Printf("Error listing %s: %v\n", ...)` to STDOUT with `continue` and final `return nil` (exit 0). Also uses `context.Background()` instead of `cmd.Context()`. `HistoryOptions.JSON` (`service.go:190`) is dead the same way.
- Why it matters: festival-app bundles dev-profile camp and `flow` is part of its surface; parsing this stdout as JSON fails 100% of the time, and failures are invisible to exit-code checks.
- Fix: emit a versioned payload via `jsoncontract` like workflow list/show; errors to stderr with non-zero exit; `cmd.Context()`.

**N-8. Intent READ commands run a filesystem migration as a side effect: `intent list/find/count/show` can rename `workflow/intents/` into `.campaign/intents/` during a `--format json` read, unlocked.**
- Where: `cmd/camp/intent/list.go:80-82` (and `find.go:71-73`, `count.go:50-52`, `show.go:69-71`) call `svc.EnsureDirectories(ctx)`; `internal/intent/migration.go:50-78` MkdirAlls seven status dirs and calls `ensureCanonicalIntentRoot` (`:81+`), which relocates the legacy intent root.
- Evidence: re-verified by read; the JSON pass also reproduced it live in a throwaway campaign (seeded legacy layout, ran `camp intent list --format json`, the legacy tree was migrated with clean JSON on stdout and nothing on stderr).
- Why it matters: reads dirty git status and mutate campaign structure; two concurrent intent reads (festival-app polls these) can race the same directory rename with no lock; this is the same "read mutates state" class as the workitem prune side effect (R18) but with a directory tree instead of one file.
- Fix: move migration to `camp init --repair` and explicit write paths; reads should at most warn that a migration is pending. Fold into R18.

**N-9. `camp workitem create` is non-atomic: it MkdirAlls the target BEFORE deriving the ref; any later failure leaves an empty directory that permanently blocks retry with "use adopt".**
- Where: `internal/commands/workitem/create.go:77-89` (stat guard and `os.MkdirAll(target)` at `:78-84`, `deriveUniqueRef` at `:86` runs a full campaign `Discover` that hard-fails on any unreadable corner).
- Evidence: re-verified by read; reproduced live by the JSON pass (unreadable `festivals/active` made create exit 1 leaving the empty dir; retry then exits 2 with `target directory already exists — use camp workitem adopt`).
- Why it matters: this is exactly the partial-success-retry failure shape festival-app just fought in FA0011 (retry must be chain-only, never re-create); camp's own create makes the app's retry path unsatisfiable without a manual adopt.
- Fix: derive id and ref before creating the directory, or remove the directory on any post-MkdirAll failure path.

### Medium

**N-10. `camp sync` silently leaves submodules at local default-branch tips while reporting success, and discards the checkout error; later ref syncs can REGRESS pointers.** `internal/sync/sync.go:250` calls `git.CheckoutDefaultBranch` discarding both return values; `validateUpdate` (`:408-428`) deliberately skips `+` drift entries with a comment claiming the drift is expected "since we just updated to the recorded commit", which is backwards: line 250 just moved HEAD OFF the recorded commit onto the local branch tip. On a machine with a stale local `main`, sync exits 0 claiming correct commits while trees sit at old code, and a subsequent refs-sync/`--include-refs` commit regresses the recorded pointers to the stale tips. Fix: fast-forward or warn when branch tip != recorded gitlink; surface checkout errors; correct validateUpdate.

**N-11. The `__manifest` agent-gating surface is blind to most of the festival-app surface and contradicts it where it does speak.** `cmd/camp/manifest.go:55` emits only commands carrying an `agent_allowed` annotation (44 entries). `doctor` is annotated `agent_allowed: false` ("Has --fix mode that is destructive", `cmd/camp/doctor.go:52-55`) even though `doctor --json` is a read surface the app shells; meanwhile `intent list/find/count/show/add`, `flow items`, `status all`, `workflow list/show`, `version`, and `concepts` are absent entirely, so an enforcement layer can neither allow the JSON reads nor flag the genuinely interactive unannotated commands (`intent add` defaults to a TUI). Fix: annotate every shipped command; split doctor's read vs `--fix` story.

**N-12. Intent promote rollback can `os.RemoveAll` a pre-existing design directory.** `internal/intent/promote/promote.go:146-178` decides `createdNow` solely by whether `README.md` existed; if `workflow/design/<slug>/` pre-existed with other files but no README, a later `Move`/`Save` failure triggers `removeCreatedDesignDoc`'s `os.RemoveAll(absDir)` (`:241-253`), deleting the user's pre-existing files. Slug collisions are realistic because `intentIDTimestampSuffix` returns empty for nonconforming IDs. Fix: track directory creation (Mkdir, not MkdirAll-and-assume); on rollback remove only what was created.

**N-13. `camp pull`/`camp push` hijack git's `-p` shorthand and can silently eat the next argument.** `internal/git/resolve.go:127-132` (`ExtractSubFlags`, used under `DisableFlagParsing`) consumes bare `-p` as `--project`: `camp pull -p` (user means `--prune`) drops the flag silently; `camp pull -p origin` consumes `origin` as a project path. No `--` terminator handling. Related: `handlePullError`'s hint `camp pull -p <display-name>` (`cmd/camp/pull.go:312`) suggests a name form `resolveProjectPath` cannot resolve. Fix: intercept only `--project`/`--project=`; honor `--`; fix the hint to `projects/<path>`.

**N-14. Root `camp commit`'s ref-exclusion promise covers staging only; the final commit is unscoped.** `cmd/camp/commit.go:137-153` stages with submodule gitlinks excluded per the help text, but the commit at `:189-200` has no `Only` scope, so refs staged earlier (prior aborted run, raw `git add`, or a concurrent fest/camp process staging in the window) are swept in under this message and tag. Distinct from GIT-2 (which is the sync sites). Fix: scope the final commit to the paths camp staged (with N-4 in mind), or detect staged `projects/*` gitlinks pre-commit and refuse without `--include-refs`.

**N-15. `camp intent list --type` silently drops every type after the first.** The flag is repeatable (`StringSliceP`) but `buildListOptions` uses `types[0]` only (`cmd/camp/intent/list.go:136-140`); multi-`--status` gets client-side filtering, multi-`--type` does not. Reproduced live by the JSON pass. festival-app filtering by several types gets silently wrong result sets. Fix: client-side multi-type filter like statuses, or reject more than one value.

**N-16. Path semantics are inconsistent across JSON surfaces, including mixed forms inside ONE payload.** workitem emits `campaign_root` plus relative paths (good); intent emits absolute, symlink-canonicalized paths (`/private/tmp/...` on macOS, so they do not even prefix-match an unresolved campaign root; `internal/intent/parse.go:142`); quest emits absolute paths including the `quest.yaml` filename (`internal/quest/types.go:56`); `status all --json` mixes both in one array, the campaign-root entry absolute (`cmd/camp/status_all.go:113-114`) while submodule entries are rewritten relative (`:169`). A consumer needs three join rules plus symlink resolution. Fix: campaign-relative paths plus one `campaign_root` field everywhere; fold into R19/R20.

**N-17. `camp concepts` has no machine mode and exits 0 with prose on stdout when outside a campaign.** `cmd/camp/concepts.go:28-34`: prints `Not in a campaign` to stdout and returns nil, unlike every other command's non-zero typed error; no `--json` exists at all despite the command being on festival-app's surface list. Fix: standard error plus a versioned `--json`.

**N-18. `camp status all` writes a cache file on every read and its `--no-cache` flag is dead.** `cmd/camp/status_all.go:117` runs `writeStatusCache` unconditionally (creating `.campaign/cache/gitstatus/status.json` during `--json` reads); `statusAllNoCache` (`:42,:50`) is never read and there is no cache-READ path anywhere, so the help text describes caching behavior that does not exist. Fix: drop the flag and the write-on-read, or implement the read path and honor the flag.

### Low / low-medium

**N-19. Second clobbering `moveFile` call site: `EditWithEditor` status changes.** `internal/intent/service.go:259` uses the same unguarded `moveFile`; additionally `moveFile`'s copy fallback (`internal/intent/helpers.go:52-60`) fires on ANY rename error, not just EXDEV, and opens dest with `os.Create` (O_TRUNC). Extends WF-5/R10: the guard must land in or around `moveFile` itself, and the fallback should be gated on `errors.Is(err, syscall.EXDEV)`.

**N-20. `camp skills` projection with `--force` silently replaces user-owned symlinks.** `internal/skills/projection.go:188-206`: the managed-link check is bypassed under force for `StateBroken`/`StateMismatch` symlinks (`os.Remove` plus relink); real files/dirs are protected, hand-made symlinks are not. Fix: treat unmanaged symlinks like `StateNotALink` conflicts or print the replaced target.

**N-21. `internal/config/allowlist.go:84` writes config with plain `os.WriteFile`.** Every sibling writer in the package uses `fsutil.WriteFileAtomically`. Add this site to the R11 sweep list (it was missing from WF-6).

**N-22. Pins migration silently drops valid pins under a symlinked campaign root.** `internal/pins/pins.go:200-235`: `MigrateAbsoluteToRelative` EvalSymlinks the pin but never the root; pins stored in resolved form under a symlinked root fail `filepath.Rel` containment and are removed as "external" on the next save, silently. Fix: resolve the root too; report dropped pins.

**N-23. "no changes added to commit" is not classified as `ErrNoChanges`.** `internal/git/errors.go:165-166` matches only "nothing to commit"; the refs-only-drift case at root `camp commit` (StageAllExcluding stages nothing, HasChanges true from pointer drift) surfaces a raw wrapped git error instead of the friendly refs-excluded hint that `camp stage` gives for the identical state. Add both git phrasings and mirror stage.go's hint.

**N-24. `camp intent add` shorthand collision and zero machine output.** `-f` means `--full` (launch TUI) on `intent add` (`cmd/camp/intent/add.go:81`) while meaning `--format` on every sibling intent command; an agent's habitual `-f json` flips TUI mode. Success output is human prose only (no way to learn the created id/path mechanically). Extends UX-9/JSON-2.

**N-25. `camp version --short` silently beats `--json`, and the JSON marshal error path exits 0.** `cmd/camp/version.go:26-38`. The one command a version-gating bootstrap is guaranteed to call. Extends UX-15/BR-4.

**N-26. Gather extension: a metadata `Save` failure after `CreateDirect` leaves the half-written gathered intent on disk while the command errors** (`internal/intent/gather/gather.go:202`); the swallow at `:211-232` was already known, this orphan-file path was not. Fold into R33.

**N-27. Caution, not user-facing today: `internal/buildutil/tasks/clean.go:46` shells `rm -rf *.bak *.old *.backup *.wip *~` in the process cwd.** Dev tool only; flag it before anything ever wires buildutil clean to a user-facing command.

### Hunted and found clean (so the festival does not re-litigate)

Intent filesystem migration guards (`migration_filesystem.go`), quest Rename/Restore rollback, attach marker handling, workitem links WithLock and QuarantineBroken, leverage reset/backfill boundaries, gendocs glob scope, project add/new creation guards, `camp sync` orphan-gitlink cleanup (index-only), shortcuts reset confirmation flow, commitkit message injection (argv-safe), `NormalizeFiles` path escaping, `ExpandTrackedPaths` rename parsing (correct in isolation; N-5 is the caller's scoping), worktree add/remove plumbing, push-all probe classification, workitem/workflow JSON family determinism and nil-guards, `jsoncontract.Requested` `--json=false` handling, quest empty-list `[]`, no ANSI bleed into piped output, no map-order nondeterminism reaching emitted JSON.

---

## 4. Conclusion: do the first-pass recommendations and phasing stand?

**The first pass survives adversarial re-verification intact: 20 of 20 re-checked findings confirmed, none refuted, none overstated.** Its R1-R36 list and four-phase shape remain valid. The fresh hunt, however, found two new data-loss paths that belong at the very top of P0 and several contract bugs that must move into the festival-app coordination phases. Required reordering:

1. **P0 additions (data loss, same tier as R2):** N-1 (`camp cp` same-path truncation), N-2 (fresh/prune dirty-worktree force removal), N-3 (project remove `.git/modules` destruction), N-6 (sync fallback RemoveAll plus non-submodule reclone). N-2 merges naturally into R16 (prune tests); N-6 should ride with R7's container-test work since both need a hostile-remote fixture.
2. **R6 must be re-scoped:** fix N-4 (`--only` worktree semantics) in `internal/git/commit.go` first or simultaneously, then scope the sync commits, then mirror the change in fest's own `commit.go:541-553` (the fest exposure is independent of `SyncSubmoduleRef`). N-5 (staged-rename corruption) belongs in the same sequence since it shares the staging plumbing.
3. **R10 widens slightly:** the guard belongs in `moveFile` (covers both CreateWithEditor and EditWithEditor) plus the EXDEV gating (N-19).
4. **R18 widens:** "reads must not write" now covers three sites, not one: workitem prune, intent EnsureDirectories migration (N-8), and the status-all cache write (N-18).
5. **festival-app contract phase (R19/R20/R26) gains:** N-7 (flow items dead --json: arguably P0 for the app), N-9 (create non-atomicity, which breaks the app's FA0011 retry model), N-11 (manifest gaps), N-15 (intent --type), N-16 (path semantics), N-17 (concepts). If the app integration festival runs before the camp P2 phase, N-7 and N-9 must be pulled forward into P0/P1.
6. **R11 sweep list gains one site:** `internal/config/allowlist.go:84` (N-21).
7. Gate framing stays LOCAL per policy: `just test` both profiles plus `go vet -tags=integration ./...` in `just lint` and a pre-push hook; no hosted CI assumed.

Net judgment: camp's machine-facing JSON discipline in the workitem/workflow families is as good as the first pass said, and the first pass's severity calibration was accurate. The second pass's main contribution is that the destructive-command long tail (cp, fresh/prune, project remove, sync fallback) is materially worse than the first pass captured, and the scoped-commit engine's core primitive (`--only`) does not have the semantics the rest of the codebase assumes. Both clusters sit directly under daily festival operation and should shape Phase 1 of the remediation festival.
