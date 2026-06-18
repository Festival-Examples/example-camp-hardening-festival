# Findings: Error Handling, Context Propagation, and Go Standards Compliance

Standards under review (campaign Go standards): project error package everywhere with wrapped errors, no `fmt.Errorf` in production code, `context.Context` first param for I/O with no ignored cancellation, no Get prefixes, dependency injection, no magic strings.

All counts in this file were produced by direct grep in this session and the headline items re-verified by reading the cited code.

---

## ERR-1. 327 `fmt.Errorf` occurrences in production code despite the mandate (HIGH)

Non-test counts by area (grep, this session): `cmd/` 111, `internal/` 195, `pkg/` 9, `tools/` 12.

`internal/errors` (camperrors) exists, is well built (typed `ValidationError`, `NotFoundError`, `CommandError` with exit-code mapping in `cmd/camp/main.go:14-17`), and is used consistently in the newer trees (`internal/commands/workitem`, `internal/commands/workflow`). The older surface mixes freely, sometimes in the same function: `cmd/camp/navigation/shortcuts.go:738` uses `fmt.Errorf("not in a campaign: %w", err)` four lines from a `camperrors.Wrap` call. Dense clusters: `cmd/camp/promote/detect.go:24-72` (12 raw occurrences in one file), `cmd/camp/dungeon/move.go:73-143`, `internal/state/location.go` (throughout), `internal/nav/index/cache.go:34,42,48` and onward, `internal/git/query.go:16` (the shared git Output helper itself wraps with fmt.Errorf, propagating non-compliance to every caller).

Notably `pkg/commitkit` (9 occurrences, e.g. `commitkit.go:193,199,209`) is the public API fest imports, so fest receives untyped errors from camp's own facade.

**Why it matters:** untyped errors defeat the exit-code mapping and any future programmatic error handling; the standard is unenforced so the count grows with every PR.
**Recommendation:** add a forbidigo (or custom) lint rule banning `fmt.Errorf` outside `tools/`, with a burn-down allowlist like the existing host-fs-test ratchet; migrate `internal/git/query.go` and `pkg/commitkit` first since they propagate the widest.

## ERR-2. No signal-aware root context: cancellation is wired but never triggered (MEDIUM)

- `cmd/camp/main.go:12` calls `Execute()`; `cmd/camp/root.go:83` runs `rootCmd.Execute()`, not `ExecuteContext`. There is no `signal.NotifyContext` at the root; the only signal wiring in the repo is `cmd/camp/leverage/backfill.go:42`.
- Consequence: `cmd.Context()` is always `context.Background()`. The codebase does an excellent job threading ctx through I/O (every traced git exec uses `exec.CommandContext`, helpers pre-check `ctx.Err()`), but nothing ever cancels it. Ctrl-C falls back to default process kill semantics, so lock-retry loops, multi-repo pulls, and container-test orchestration cannot clean up.
- Defensive `if ctx == nil { ctx = context.Background() }` fallbacks are scattered through cmd files (`cmd/camp/commit.go:75,94`, `cmd/camp/sync.go:93`, `cmd/camp/doctor.go:86`, `cmd/camp/status_all.go:85`, `cmd/camp/stage.go:63`, all of `cmd/camp/worktrees/*`), which both masks the gap and adds noise.
- **Recommendation:** in main, `ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)` and `rootCmd.ExecuteContext(ctx)`; delete the nil-ctx fallbacks (cobra guarantees a non-nil context under ExecuteContext).

## ERR-3. Cancellation-ignoring sleep in the git retry loop (LOW)

- `internal/git/retry.go:159`: `time.Sleep(backoff)` between lock-retry cycles ignores ctx for up to ~2s. Everything around it is ctx-aware.
- **Recommendation:** `select { case <-ctx.Done(): return ctx.Err(); case <-time.After(backoff): }`.

## ERR-4. Helper-created contexts deep in call stacks (LOW)

- `cmd/camp/root.go:119` (`loadShortcutsForExpansion`), `cmd/camp/navigation/go.go:453`, `cmd/camp/run.go:199`, `internal/complete/complete.go:164`, `cmd/camp/plugin.go:53`, `cmd/camp/pins.go:210` create fresh `context.Background()` instead of receiving one. Mostly pre-Execute argv rewriting where no command context exists yet, so severity is low, but once ERR-2 lands these should accept the root ctx.

## ERR-5. Silently swallowed errors at state-mutation sites (MEDIUM)

Verified sites where an error from a write or cleanup is discarded with `_ =` or an empty branch:

- `internal/scaffold/init.go:388`: `_ = config.SaveRegistry(ctx, reg)` during `camp init`. A freshly initialized campaign can silently fail to register; the user discovers later when switch/navigation cannot find it. This is the worst one.
- `cmd/camp/switch.go:98`: `_ = config.SaveRegistry(ctx, reg)` (last-access update; acceptable to degrade but should warn).
- `cmd/camp/project/add.go:248`: registry save error dropped in the resolver path.
- `cmd/camp/init/command.go:289`: `festInitialized, _ = initializeFestivals(ctx, ...)`; a failing `fest init` produces a success banner with a generic "run manually" skip and no cause (`internal/scaffold/init.go:360-362`).
- `internal/nav/index/cache.go:224-227` and `:247-249`: cache load/save errors vanish into empty branches with literal comments "in production this would be logged". A permanently unwritable cache dir means a silent full index rebuild on every invocation.
- `internal/project/add.go:261,266`: submodule-add cleanup steps 3 and 4 (`_ = rmSection.Run()`, `_ = rmCached.Run()`); a failed `.gitmodules` cleanup makes the next `camp project add` of the same name fail with a confusing "already exists in index".
- `internal/scaffold/repair.go:784-786`: best-effort `os.RemoveAll(m.Source)` after migration, error discarded; repair keeps re-reporting the stale legacy dir.
- `cmd/camp/worktrees/clean.go:240-243`: `git.Remove` failure swallowed entirely before unconditional `os.RemoveAll` (this one is a data-loss bug, covered as GIT-1 in findings-subsystems.md).

**Recommendation:** registry-save failure during init must return the error; the rest should at least warn on stderr. Consider an errcheck lint pass with a small allowlist.

## ERR-6. Get-prefix and naming standard violations (LOW)

- `internal/git/config.go:24,35`: `GetUserEmail`, `GetUserIdentity` (both also dead code per ARCH-5).
- Sweep for remaining `Get*` exports while deleting dead code.

## ERR-7. Magic strings for cross-cutting concepts (MEDIUM)

- Status/dir names (`inbox`, `active`, `ready`, `dungeon`, `completed`, `archived`, `someday`) and workflow type names (`intent`, `design`, `explore`, `festival`) appear as raw literals across packages rather than shared constants; example cluster: porcelain/status literals in `cmd/camp/status_all.go:216-230`, status dirs in dungeon and intent services. The workitem JSON layer does define enums properly (`internal/workitem/model.go:34-52` documents the lowercase enum), but producers elsewhere hand-write the same strings.
- `internal/git/run.go:56` hardcodes `"index.lock"` while `cmd/camp/pull.go:126-131` resolves the real gitdir lock path; the two disagree for worktrees and submodules.
- **Recommendation:** centralize status/type vocabularies in one constants package per domain; replace the hardcoded index.lock path with `ResolveGitDir` (`internal/git/lock.go:89-127`).

## ERR-8. Error classification is substring-based and locale-fragile (LOW)

- `internal/git/errors.go:159-181`: `ClassifyGitError` matches `nothing to commit` but not `no changes added to commit`; any stderr containing the word "submodule" classifies as `GitErrorSubmodule`. Nothing sets `LC_ALL=C` on git commands, so non-English locales break classification.
- **Recommendation:** set `LC_ALL=C` (or `LANG=C`) in the shared git exec helpers; tighten the submodule match.

## ERR-9. What is compliant (informational)

- Context-first signatures are near-universal for I/O; `exec.CommandContext` is used for every git invocation traced (51 non-test files); many helpers pre-check `ctx.Err()`.
- `exec.Command` without ctx is confined to `internal/buildutil` (the build tool, a defensible exemption that should be documented as such) and `internal/intent/tui/viewer_actions.go:68-91` (fire-and-forget `open`/`xdg-open`, acceptable).
- `cmd/camp/main.go:11-21` exit-code mapping via `camperrors.CommandError` is a good pattern; the problem is the bypasses (UX-14) and untyped errors (ERR-1), not the mechanism.
- Stderr capture on commit paths is good (`CombinedOutput` surfaced through `camperrors.NewGit`); weaker on `internal/git/query.go:14-16` (`cmd.Output()`, stderr discarded).
