# Findings: Repository and Package Architecture

**Repo:** `projects/camp` at commit `5c22d2b` (2026-06-10)
**Scale:** 917 Go files, ~186k lines total, ~94.9k non-test production lines, 23.6k of them under `cmd/`.
**Build health:** `go build ./...`, `go build -tags dev ./...`, and `go vet ./...` all pass. Dead-code analysis was run with `golang.org/x/tools/cmd/deadcode` against both stable and dev tag sets; only symbols dead in BOTH builds are reported below.

---

## ARCH-1. Two parallel cobra command trees with no convention (HIGH)

There are two registration styles with no documented rule for which a new command should use:

- **Tree A:** `cmd/camp/` top-level files plus 15 subpackages (`navigation`, `intent`, `project`, `dungeon`, `leverage`, `quest`, `attach`, `init`, ...), registered via package-level `var` plus scattered `init()` `rootCmd.AddCommand(...)` calls (e.g. `cmd/camp/commit.go:64`, `cmd/camp/pull.go:49`).
- **Tree B:** `internal/commands/{flow,fresh,release,workflow,workitem}` using constructor functions (`NewWorkitemCommand()` etc.), wired in `cmd/camp/root.go:211-216` and via `release.Register(rootCmd)` at `root.go:224` (which adds `fresh`, `build-profile`, and in dev builds `flow` via `internal/commands/release/register_dev.go`).

Tree B is the newer, better style (testable, no package-level mutable flag vars) and is where the highest-discipline code lives (jsoncontract envelopes, camperrors). There is also cross-tree coupling: `cmd/camp/commit_workitem.go:7` imports `internal/commands/workitem` to call `EnsureRefForCommit`, i.e. a Tree A command depends on a Tree B command package for domain behavior that belongs in `internal/workitem`.

**Why it matters:** every new command forces an undocumented style decision; the trees drift in error handling, JSON contracts, and exit semantics (see findings-command-ux.md).
**Recommendation:** declare Tree B's constructor style canonical, document it, migrate Tree A incrementally, and move `EnsureRefForCommit` into `internal/workitem`.

## ARCH-2. Business logic stranded in `cmd/` (HIGH)

The cmd layer is supposed to be thin wiring. Worst offenders, verified by reading the files:

| File | Lines | Logic that should live in internal/ |
|---|---|---|
| `cmd/camp/pull.go` (423) | ~330 | raw `exec.CommandContext(ctx, "git", ...)` with lock retry at `pull.go:91-117`, error classification `classifyPullCommandError` at `pull.go:119-153`, full pull-all engine `runPullAll`/`pullSingleTarget` at `pull.go:170-305`, rebase abort at `pull.go:306-314` |
| `cmd/camp/status_all.go` (411) | ~340 | `getRepoStatus` (`:176-257`) shells out to git, `renderStatusTable` (`:285-384`), cache writer `writeStatusCache` (`:391+`) |
| `cmd/camp/navigation/shortcuts.go` (890) | most | shortcut diff/reset engine: `computeShortcutDiff` (`:617-656`), `runShortcutsReset` (`:731-852`), `runShortcutsAddJump` (`:477-581`), `isAutoShortcut` (`:593`). `internal/shortcuts` exists but is imported only by this file, and its own `Expander`/`FeedbackWriter` are dead (ARCH-5) |
| `cmd/camp/navigation/go.go` (540) | ~95 | `handleRelativePathNavigation`, `resolveRelativePathNavigation`, `fuzzyResolveDirectory` (`go.go:337-432`) are core resolution semantics |
| `cmd/camp/leverage/helpers.go` (385) | ~100 | `computeProjectScore` (`:256-321`) is the effort/person-month scoring policy; `persistCurrentSnapshots` (`:322-360`) is snapshot persistence |
| `cmd/camp/dungeon/crawl.go` (463) | ~260 | commit-path computation and staging policy: `commitCrawlChanges` (`:165`), `stageTrackedCrawlSourceDeletions` (`:325-399`), `isSafeCrawlCommitPath` (`:419`) |
| `cmd/camp/init/command.go` (415) | ~200 | `RunFlow` (`:151-353`), skill projection `projectCampaignSkills` (`:383+`) |
| `cmd/camp/intent/add.go` (408) | ~100 | local resolver type `intentAddCampaignResolver` (`:267-331`), capture orchestration (`:350-376`) |
| `cmd/camp/run.go` | ~70 | duplicated shortcut/sub-shortcut resolution at `run.go:96-165`, including a redundant `index.Resolve` probe at `:121-130` before the real resolve at `:133` |

**Why it matters:** this logic is untestable without building the binary, unusable from other commands (run.go already had to re-implement go.go's resolution), and several of the worst correctness bugs in this review live precisely in these files (pull rebase abort, status porcelain miscount, worktrees clean data loss).
**Recommendation:** extract into `internal/pull` (or `internal/git`), `internal/status`, `internal/shortcuts`, `internal/nav`, `internal/leverage`, `internal/dungeon` respectively. The pattern of good delegation already exists (`cmd/camp/commit.go` delegating to `internal/git` and `pkg/commitkit`).

## ARCH-3. flow vs workflow naming inversion (MEDIUM)

Three crossing names for three different things:

- `internal/workflow` is the status-workflow domain (`.workflow.yaml`, transitions, dungeon paths).
- `internal/flow` is a flow registry/runner over `.campaign/flows/registry.yaml` (`internal/flow/registry.go:1-2`).
- The CLI command `camp flow` (`internal/commands/flow/flow.go:17-19`, aliased `wf`) is mostly backed by `internal/workflow` (7 of its files import it), with `internal/flow` used only by `run.go`/`list.go`.
- The CLI command `camp workflow` (`internal/commands/workflow/workflow.go:6-13`) manages workflow collections under `workflow/<type>/`, a third concept.

**Why it matters:** an engineer (or agent) navigating from command name to backing package lands in the wrong place every time. This gets worse if `flow` ships to stable (it is dev-only today but the scaffolded skills already document it, see findings-build-release.md).
**Recommendation:** rename either the packages or the commands so command name matches backing package, before `flow` reaches stable. At minimum add package docs cross-referencing each other.

## ARCH-4. pkg/commitkit is simultaneously fest's public API and camp's internal convenience layer (MEDIUM)

The git layering itself is sound (verified, no duplicated implementations across the layers):

- `internal/git`: primitives (`Commit()` with lock retry at `internal/git/commit.go:30-44`, branches, submodules, tags).
- `internal/git/commit`: campaign-tagged commit workflows (`doCommit` formats `[OBEY-CAMPAIGN-{id}] Action: Subject`, `internal/git/commit/commit.go:33-52`), with per-concept entry points (intent, quest, workitem, crawl, project, repair). 25 cmd files import it.
- `pkg/commitkit`: explicit public facade "so that external tools (e.g. fest) can import them without depending on internal implementation paths" (`pkg/commitkit/commitkit.go:1-8`). fest consumes it (verified in `projects/fest/internal/commands/commit/commit.go` plus 2 more files).

The blur: camp's own commands import all three layers. `cmd/camp/commit.go:12-14` imports `internal/git` AND `pkg/commitkit` in the same file. Meanwhile 13 commitkit exports (`StageAll`, `Commit`, `CommitAll`, `SyncSubmoduleRef`, etc., `pkg/commitkit/commitkit.go:103-186`) are unreachable from the camp binary itself, so their only consumer is fest, with no in-repo guard that camp will not break them. Note `SyncSubmoduleRef` also carries a correctness bug that directly affects fest (findings-subsystems.md GIT-2).

**Recommendation:** treat `pkg/commitkit` as external-only (camp code uses `internal/git` and `internal/git/commit`), and add an API-compat test or doc marker declaring it fest's contract surface.

## ARCH-5. ~216 dead symbols, including three entire dead feature clusters (HIGH)

Symbols dead in BOTH stable and dev builds (deadcode tool, excluding `pkg/commitkit`'s 13 fest-facing exports and test fakes):

- `internal/nav`: 27 symbols. The entire `internal/nav/tui` picker (`Pick`, `PickScoped`, `PickPath`, `picker.go:51-172`), the whole `Query` API (`internal/nav/index/query.go:38-240`), `Builder.BuildWithOptions` (`builder.go:229`), `JumpToPath` (`direct.go:127`). Navigation moved to `go-fuzzyfinder` and left this behind.
- `internal/clone`: 24 symbols. The complete validation framework is unreachable: `NewValidator`, `DefaultValidator.{RegisterCheck,Validate,Checks}` (`validation.go:55-105`), all three check types (`check_commits.go`, `check_initialized.go`, `check_urls.go`), and all of `validation_output.go` (lines 10-162). An entire feature never wired to a command.
- `internal/concept/service.go:385-680`: `FSService` and all 9 of its methods are unreachable (`NewFSService` at `service.go:392`). ~295 lines, nearly half of a 680-line file; `DefaultService` (line 23) is the live implementation.
- `internal/shortcuts`: 13 symbols. `Expander` (`expand.go:25-105`) and `FeedbackWriter` including a levenshtein implementation (`feedback.go:20-206`) are fully dead.
- `internal/git`: 17 symbols including `MergedBranches` (`branches.go:151`), `SubmoduleExecutor` (`executor.go:203-226`), `GetUserEmail`/`GetUserIdentity` (`config.go:24,35`, which also violate the no-Get-prefix standard), `NewLockError`/`NewStaleLockError`/`NewActiveLockError` (`errors.go:184-200`).
- `internal/sync`: 8 symbols, notably four whole preflight checks `CheckUncommittedChanges`/`CheckUnpushedCommits`/`CheckURLMismatches`/`CheckDetachedHEADs` (`preflight.go:121-268`).
- Misc: `internal/campaign` cache/detect variants (`cache.go:68-85`, `detect.go:169-196`), `internal/workflow` `Service.Crawl` (`service_crawl.go:9`), `DetectTransition` (`transition.go:26`), `internal/config` 13 symbols including the whole allowlist config (`allowlist.go:39-108`), `cmd/camp/dungeon/crawl.go:307`, `cmd/camp/pins.go:281`, `cmd/camp/run.go:198`, `internal/errors` dead constructors `NewIO` (`errors.go:110`), `NewConfig` (`errors.go:84`), `NewPermission` (`errors.go:136`), `As`/`Unwrap` (`wrap.go:56,59`).
- `internal/crawl/fake_prompt.go`: a test fake shipped in a production file (6 dead methods). Move to a `_test.go` file.

**Why it matters:** roughly 2,000+ lines of unmaintained code that reads as live, misleads contributors and agents, and inflates one over-500-line file.
**Recommendation:** delete the clusters in one sweep; this is the highest payoff-to-risk item in the whole review.

## ARCH-6. 14 production files over the 500-line standard (MEDIUM)

| Lines | File |
|---|---|
| 1000 | internal/intent/tui/add_model.go |
| 890 | cmd/camp/navigation/shortcuts.go |
| 800 | internal/scaffold/repair.go |
| 719 | internal/dungeon/docs_browser.go |
| 680 | internal/concept/service.go (drops to ~385 after deleting dead FSService) |
| 677 | internal/tui/vim/editor.go |
| 618 | internal/prune/prune.go |
| 592 | internal/quest/service.go |
| 592 | internal/leverage/authors.go |
| 569 | internal/intent/tui/explorer/actions.go |
| 540 | cmd/camp/navigation/go.go |
| 530 | internal/tui/vim/motions.go |
| 520 | internal/tui/vim/buffer.go |
| 501 | internal/intent/service.go |

The top three coincide with other findings (shortcuts logic-in-cmd, repair migration safety, concept dead code), making them the best refactor targets.

## ARCH-7. Import hygiene: clean upward, leaky sideways (MEDIUM)

- No layering violation upward: zero imports of `cmd/` from `internal/` or `pkg/` (grep verified).
- `internal/ui` (presentation) leaks into domain packages: `internal/project/output.go`, `internal/prune/prune.go`, `internal/shortcuts/feedback.go`, `internal/workitem/tui/styles.go`. `internal/commands/fresh/fresh.go:25-29` bakes lipgloss styles into the command package. Domain output becomes terminal-coupled and untestable as data.
- **Recommendation:** domain packages return structs; rendering stays in cmd/ui.

## ARCH-8. Four path/fs packages: justified but unsignposted (LOW)

Verified, no actual duplication:

- `internal/paths`: config-driven campaign directory resolution (`internal/paths/resolver.go:1-30`), 40 importers, the workhorse.
- `internal/pathutil`: containment/boundary validation with symlink resolution and macOS `/var` handling (`internal/pathutil/boundary.go:10-50`, `submodule.go:10-25`), 10 importers.
- `internal/pathsafe`: a single path-segment validation regex (`internal/pathsafe/segment.go:9-26`), 3 importers.
- `internal/fsutil`: `WriteFileAtomically` (`atomic_write.go:10-14`) and `AcquireFileLock` (`lock.go:19-23`), 13 importers. Distinct from `internal/git/lock.go` (git index.lock retry).

**Recommendation:** merge `pathsafe` into `pathutil`; add one-line package docs cross-referencing the split.

## ARCH-9. External dependencies: contained but unpinned to tags (LOW)

- `github.com/lancekrogers/guild-scaffold`: exactly 2 production importers, `internal/scaffold/init.go:21` and `internal/scaffold/repair.go:18`. Fully contained. Pinned at a pseudo-version (`v0.0.0-2026...`), no tagged release.
- `github.com/Obedience-Corp/obey-shared v0.4.4-0.2026...`: 3 production files (`internal/contract/entries.go:7`, `internal/scaffold/init.go:19`, `internal/editor/editor.go:15`). Contained, also a pre-release pseudo-version.

**Recommendation:** tag both upstreams and pin tags for reproducibility.

## ARCH-10. Suspect packages checked and cleared (informational)

All five packages investigated as possibly orphaned are wired and live:

- `internal/leverage` backs the entire `cmd/camp/leverage` tree (9 importers, 1 dead symbol: `ProjectActualPersonMonths`, `authors.go:520`).
- `internal/crawl` is a deliberate shared crawl-loop primitive (package doc `types.go:1-11`), consumed by intent crawl, dungeon crawl, and triage. Good design.
- `internal/transfer` backs `camp copy`/`camp mv`.
- `internal/plugin` backs git-style plugin dispatch (`cmd/camp/plugin.go`, invoked from `root.go:76`).
- `internal/contract` declares camp's watcher-contract entries for the obey daemon, consumed by `internal/scaffold/init.go` (camp init writes `.campaign/watchers.yaml`).

One nuance: in stable builds the dev-only `flow` command means `internal/commands/flow` (1,164 lines) is compiled but unreachable. Expected per channel design, informational only.
