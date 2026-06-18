# Findings: Core Subsystems Deep Dive

Sections: registry/workspace, init/repair, project intake/linking, navigation, camp run, shell integration, git/commit subsystems, then intents/workitems/quests/dungeon/workflow.

Festival boundary check (important context fact): no camp code was found writing into fest-owned `festivals/` files. Camp scaffolds festivals only by delegating to `fest init` (`internal/scaffold/init.go:360-362`, `cmd/camp/init/command.go:289`). The boundary is respected.

---

# A. Workspace / Registry (internal/config, internal/state)

Storage model: registry at `~/.obey/campaign/registry.json` (XDG and `CAMP_REGISTRY_PATH` overrides, `internal/config/registryfile/registryfile.go:25-34`). Save path uses `fsutil.WriteFileAtomically` (`internal/config/registry.go:81`, verified), which is a correct same-dir temp file, chmod, fsync, rename (`internal/fsutil/atomic_write.go:14-48`). Single-write crash safety is good.

## REG-1. No locking around registry load-modify-save: concurrent camp processes lose updates (HIGH)
- `internal/config/registry.go:26-86` (LoadRegistry/SaveRegistry) and callers (`cmd/camp/register.go:121,164`, `internal/clone/register.go:32-43`, `internal/scaffold/init.go:384-388`) do unguarded read, mutate in memory, atomic write. Verified: `fsutil.AcquireFileLock` exists (`internal/fsutil/lock.go:22`) but its only consumers are `internal/workitem/links/save.go` and `internal/commands/workitem/doctor_ref_backfill.go`; grep of registry.go shows zero lock usage.
- **Why it matters:** concurrent camp commands are the norm in agent loops. Two simultaneous registrations both load the same snapshot; the second save erases the first. Atomic rename prevents corruption but not lost updates.
- **Recommendation:** a `config.UpdateRegistry(ctx, func(*Registry) error)` helper that takes `fsutil.AcquireFileLock(registryPath+".lock")`, re-loads inside the lock, mutates, saves. Migrate all mutating callers.

## REG-2. Stale-lock removal in fsutil has a TOCTOU race and a hard 30s assumption (MEDIUM)
- `internal/fsutil/lock.go:71-86`: `removeStaleLock` does os.Stat then os.Remove with no atomicity. Between stat (looks stale) and remove, another process can delete the stale lock and create a fresh one; this process then removes the fresh lock and two holders proceed. Any legitimate operation holding the lock past `staleLockAfter = 30s` gets its lock stolen.
- **Recommendation:** steal via rename to a unique name so only one stealer wins, or write PID and verify liveness, or move to flock.

## REG-3. Navigation state file written non-atomically and without lock (LOW)
- `internal/state/location.go:105`: `os.Create(stateFilePath)` then a loop of writes (truncate-then-write). Concurrent `cgo` invocations can interleave or truncate `state.jsonl`. `LoadHistory` hard-fails on any corrupt line (`location.go:61-63`), so a half-written file breaks the `cgo` toggle until cleared.
- **Recommendation:** build the buffer, use `fsutil.WriteFileAtomically`; treat unparsable lines as skippable.

## REG-4. Swallowed registry saves (rolled into ERR-5; the init-time one is the priority)

---

# B. Campaign Init and Repair (internal/scaffold, cmd/camp/init)

Posture is good overall: repair computes a plan first, prints a diff, requires confirmation unless `--yes` (`cmd/camp/init/command.go:216-253`); scaffold file writes are create-only (no user-file overwrite found); default-location resolution (`internal/config/campaigns_dir.go:18-47`) handles tilde expansion, relative-joins-HOME, Clean.

## INIT-1. ExecuteMigrations aborts mid-sequence and the partial result is auto-committed (HIGH)
- Verified this session. `internal/scaffold/repair.go:766-771`: per-item `os.Rename(src, dst)`, return on first error, no pre-validation pass.
- `cmd/camp/init/command.go:262-268`: the error is downgraded to a warning line (`Migration error: %v`), `migrationCount = moved`, and then `commitRepairChanges(...)` at `command.go:282-284` runs unconditionally for `p.Repair && !p.DryRun`, committing the half-migrated tree as a successful repair.
- Mitigating factor: directory `os.Rename` onto an existing non-empty dir fails rather than overwrites, so this is split-state risk, not overwrite-loss risk. But these migrations move user content (legacy workflow/intents layouts), and a rerun can fail confusingly when a destination name now exists.
- **Recommendation:** pre-validate every (src exists, dst absent) pair before moving anything; on mid-sequence error skip the auto-commit, propagate a non-zero exit, and print recovery guidance.

## INIT-2. Best-effort cleanups hide failures (LOW)
- `internal/scaffold/repair.go:784-786`: post-migration `os.RemoveAll(m.Source)` error discarded (stale legacy dir re-reported forever).
- `internal/scaffold/repair.go:558-570`: .gitignore append is read-modify-write via plain `os.WriteFile`, not the atomic helper.
- `cmd/camp/init/command.go:289` plus `internal/scaffold/init.go:360-362`: `fest init` failure is invisible; user gets the "Campaign Initialized" banner with a generic skip note and no cause. Distinguish "fest not installed" from "fest init failed" and print the error.

---

# C. Project Intake and Linking (internal/project, internal/clone, internal/attach)

Mechanism: `camp project add <url|--local>` adds a git submodule via `internal/project/add.go`; `camp project link` (`internal/project/link.go` AddLinked) creates a symlink in `projects/` plus a `.camp` marker in the target. All git invocations use argv arrays (no shell), so there is no injection in intake. Submodule partial failure is handled unusually well: `cleanupFailedSubmoduleAdd` (`add.go:243-267`) removes the checkout, `.git/modules` entry, `.gitmodules` section, and the staged gitlink. Duplicate link targets are blocked via `EvalSymlinks` on both sides (`link.go:243-278`), correctly handling macOS `/var` vs `/private/var`.

## LINK-1. Marker/symlink ordering leaves a crash window (LOW)
- `internal/project/link.go:126-140`: symlink created at :126, marker written at :137; marker-write failure removes the symlink (correct), but a crash between the two leaves a symlink whose target has no `.camp` marker. Duplicates are still blocked (uniqueness scans symlinks, not markers, `link.go:115`), so the residual effect is only a lost "linked to campaign X" stamp.
- **Recommendation:** write the marker first, remove it on symlink failure. Strictly better at no cost.

## LINK-2. Quiet failure modes (LOW)
- `internal/project/add.go:261,266`: cleanup steps swallow errors (cross-ref ERR-5).
- `internal/project/link.go:375-378`: `normalizeCampaignRoot` silently falls back to the unresolved path when EvalSymlinks fails; the same campaign root recorded in two shapes can make `markerMatchesCampaign` (`link.go:330-341`) produce a spurious "already linked to another campaign".
- `internal/project/list.go:41-51`: broken link symlinks are skipped with `continue` and no warning; a moved or deleted target makes the project vanish from `camp project list` silently while the dead symlink stays in `projects/`.

---

# D. Navigation (cmd/camp/navigation, internal/nav, internal/nav/index)

Mechanism: `cgo` wraps `camp go --print`. Resolution: shortcut parse, category mapping, then `index.Resolve` against `.campaign/cache/nav-index.json`. Invalidation (`internal/nav/index/cache.go:96-168`) is mtime-based and fairly thorough (24h max age, index version, camp binary mtime, .gitmodules, campaign.yaml, projects/ dir mtime, worktrees, depth-bounded workflow/intents/festivals mtimes). Cache writes are atomic (`cache.go:51`).

## NAV-1. Resolved paths printed without existence check; linked-project deletion defeats mtime invalidation (MEDIUM)
- `cmd/camp/navigation/go.go:273-277`: under `--print` the resolved path is printed with no `os.Stat`. Same for pins at `go.go:533-539`. Deleting `projects/<x>` bumps the projects/ mtime so most staleness is caught, but a linked project whose external target is deleted leaves the symlink and the mtime unchanged, and deep changes inside the depth-bounded dirs can go stale within the 24h window.
- **Why it matters:** `cd "$(camp go --print ...)"` lands on a nonexistent directory with a confusing shell error.
- **Recommendation:** stat the final path before printing; on miss, force one rebuild (`GetOrBuild(..., true)`) and retry, then error cleanly.

## NAV-2. --print stream discipline (MEDIUM)
- `cmd/camp/navigation/go.go:256-258`: the `--list` branch runs before the `--print` branch and prints human help to stdout, so `--list --print` emits non-path text on stdout.
- `go.go:265-271`: the multi-match disambiguation warning is gated `!printOnly` even though it already writes to stderr; with `--print`, scripts get a silent best-match choice and no ambiguity signal.
- **Recommendation:** reject `--list` with `--print`; drop the `!printOnly` gate.

## NAV-3. Concurrent index rebuilds unserialized, cache errors swallowed (MEDIUM)
- `internal/nav/index/cache.go:216-252`: no lock around stale-check/rebuild/save; two camp processes rebuild redundantly. Load error (:224-227) and save error (:247-249) vanish into empty branches (cross-ref ERR-5).
- **Recommendation:** `fsutil.AcquireFileLock` around rebuild plus save; warn on save failure.

## NAV-4. Resolution logic in cmd/ and duplicated in run.go (cross-ref ARCH-2)
- `go.go:337-432` resolution helpers and shortcuts.go's diff/reset engine belong in `internal/nav` / `internal/shortcuts`; `cmd/camp/run.go:96-165` re-implements ~70 lines of shortcut and sub-shortcut resolution including a redundant double `index.Resolve`.

## NAV-5. Sub-shortcut paths joined without validation (LOW)
- `internal/nav/index/target.go:37`: `filepath.Join(t.Path, subPath)` accepts `..` and absolute values, no existence check. Config is user-owned so hygiene not security; a typo silently navigates outside the project. Reject `..` and absolute at config load.

---

# E. camp run / cr (cmd/camp/run.go, cmd/camp/cmdutil/command.go)

Mechanism: detects root via `campaign.DetectCached`, supports `@shortcut` working-dir selection, exact-match project dispatch to `just`, else raw shell from root. Child exit codes propagate correctly end to end: `camperrors.NewCommand(fullCmd, exitErr.ExitCode(), ...)` (`cmdutil/command.go:38-40`) to `os.Exit(cmdErr.ExitCode)` (`cmd/camp/main.go:14-17`). Env handling is sane (inherits environ plus `CAMPAIGN_ROOT`, stdio passthrough).

## RUN-1. Argument quoting destroyed by strings.Join plus sh -c, including the just dispatch path (HIGH)
- Verified this session. `cmd/camp/cmdutil/command.go:22-27`:

```go
fullCmd := cmdStr
if len(extraArgs) > 0 {
    fullCmd = fmt.Sprintf("%s %s", cmdStr, strings.Join(extraArgs, " "))
}
cmd := exec.CommandContext(ctx, "sh", "-c", fullCmd)
```

- and `cmd/camp/run.go:183` routes the project just-dispatch path through this (`ExecuteCommand(ctx, "just", projectDir, root, commandArgs[1:])`); `run.go:161,192` also flatten with `strings.Join`.
- For the raw-command form, shell interpretation is documented behavior (fine). But the just-dispatch path promises argv semantics and then flattens them: `cr camp commit -m "fix: two words"` becomes `sh -c "just commit -m fix: two words"`, splitting the message. Any arg containing spaces, `$`, `;`, quotes, or globs is corrupted. The `cr` shell alias funnels everything here, so this bites daily workflows.
- **Recommendation:** for project dispatch, exec just directly with argv (`exec.CommandContext(ctx, "just", args...)`, `cmd.Dir = projectDir`). Keep `sh -c` only for the explicit raw-command form, or shell-quote each arg when joining.

---

# F. Shell integration (internal/shell, cmd/camp/shell_init.go)

Quoting of `"$dest"`, `"$@"`, `cd "$dest"` is correct in bash and zsh templates; no eval of untrusted content. Template parity across zsh/bash/fish confirmed.

## SH-1. go wrapper discards camp stderr and replaces real errors with a generic message (MEDIUM)
- `internal/shell/templates/bash.sh.tmpl:33` (and :24; `zsh.sh.tmpl:25,35`): `dest=$(command camp go "$@" --print 2>/dev/null)` then on empty output `echo "camp: not found: $*"`.
- `--print` already keeps stdout path-only, so `2>/dev/null` only hides camp's diagnostics (including the disambiguation list go.go sends to stderr, and config errors). Exit code is never checked, only string emptiness.
- **Recommendation:** drop `2>/dev/null`, check `$?`, keep the generic message only when camp produced no output at all.

## SH-2. --print passthrough exists for workitem but not go/switch (MEDIUM)
- `bash.sh.tmpl:46-56` passes through when the user supplies `--help/--json/--print` to `workitem`, but the `go|g` and `switch|sw` branches have no passthrough. Inside an init'd shell, `camp go p x --print` gets intercepted (the function cds, prints nothing), so `cd "$(camp go ... --print)"` in a sourced script silently receives empty.
- **Recommendation:** mirror the workitem passthrough scan in the go and switch branches.

## SH-3. Latent escaping gaps in generated completions (LOW)
- `internal/shell/shortcuts.go:63-66` (fish `-d "%s"`) and `:43` (zsh `'%s:%s'` where `:` is the `_describe` separator) interpolate descriptions without escaping. Data source today is the static `config.DefaultNavigationShortcuts()` table, so latent, not exploitable; the first default description containing a quote or colon breaks the eval'd init script for every user. Also: completions cover only default shortcuts, not user-defined ones; bash `COMPREPLY=($(compgen -W ...))` (`bash.sh.tmpl:113,128`) word-splits candidates containing spaces.
- **Recommendation:** per-shell escaping; document the no-spaces constraint or build COMPREPLY with while-read.

---

# G. Git and Commit Subsystems

How the flows work (verified): `camp commit` (`cmd/camp/commit.go:91-204`) detects root, `git.ResolveTarget` picks root/sub/project, auto-stages (`-a` default true), at root excludes submodule gitlinks via `git.StageAllExcluding` pathspecs (`internal/git/commit.go:318-332`). The `[OBEY-CAMPAIGN-...]` tag comes from `config.LoadCampaignConfig(ctx, campRoot).ID` threaded through `commitkit.PrependContextTagsFull` (`commit.go:182-185`) into `internal/git/campaign_tag.go:17-42` (ID truncated to 8 chars; the `WI-WI-<hex>` double prefix is intentional and test-pinned). `camp p commit` (`cmd/camp/project/commit.go:64-187`) resolves the project by flag or longest-path cwd match, stages all in the submodule, commits, optional `--sync` stages the gitlink at root. The strongest code in the subsystem is `internal/git/commit/commit.go` (scoped `git commit --only` over `ExpandTrackedPaths`, which parses `-z` rename records correctly, `internal/git/commit.go:250-306`) and the cycle-based `WithLockRetry`.

Multi-repo ops (`push_all`, `pull_all`, `status_all`) are strictly sequential with per-iteration ctx checks; no goroutines, no shared-output or registry races found. Status cache write is atomic (`status_all.go:391-411`).

## GIT-1. `camp worktrees clean` can delete uncommitted work, ignoring --force (HIGH)
- Verified this session. `cmd/camp/worktrees/clean.go:240-247`:

```go
if err := git.Remove(ctx, result.path, force); err != nil {
    // Fall through to filesystem removal
}
// Remove the directory
if err := os.RemoveAll(result.path); err != nil {
```

- The `git worktree remove` failure (which is git refusing because the tree is dirty) is swallowed and `os.RemoveAll` runs unconditionally, regardless of `--force`. The staleness heuristic (`checkWorktreeStale`, `clean.go:189-226`) also classifies ".git is a directory (not a worktree)" and "missing .git file" as stale, so a full clone or plain data directory parked under `projects/worktrees/<proj>/` (enumerated by `ListProjectWorktrees`, `internal/worktree/paths.go:116-133`) gets rm-rf'd with all uncommitted and unpushed content. Contrast `camp project worktree remove` (`cmd/camp/project/worktree/remove.go:90`) which correctly lets git refuse. The package has zero tests (TEST-4).
- **Recommendation:** only `RemoveAll` for the "gitdir target does not exist" reason; never auto-remove `.git`-directory entries; gate any non-empty directory removal behind `--force` plus a dirty check.

## GIT-2. Parent-ref sync commits sweep unrelated pre-staged changes (HIGH)
- `cmd/camp/project/commit.go:196-207` (`syncParentRef`): stages the submodule relPath then commits with no `Only` scope. Anything the user already had staged at the campaign root is committed under "update X submodule ref".
- `pkg/commitkit/commitkit.go:186-213` (`SyncSubmoduleRef`, the public API fest calls): verified this session, it DOES stage only the submodule ref (`git.StageFiles(ctx, campaignRoot, projectRelPath)`), but the no-op guard `git.HasStagedChanges(ctx, campaignRoot)` (:197) checks the whole repo and the `git.Commit` (:208) is unscoped. So pre-staged unrelated files both defeat the no-op check and get swept into the sync commit. This ships wrong-content commits through fest too.
- `camp refs-sync` got this right with a staged-clean precheck (`cmd/camp/refs/commands.go:57-62`).
- **Recommendation:** commit with `CommitOptions.Only: []string{projectRelPath}` (support exists, `internal/git/commit.go:62-65`) in both places, or adopt the refs-sync precheck. Treat the commitkit fix as a fest-facing contract change.

## GIT-3. `camp pull all` aborts rebases it did not start (HIGH)
- Verified this session. `cmd/camp/pull.go:305-314` (`handlePullError`): on any pull failure, if `git.IsRebaseInProgress(ctx, t.path)` then `_ = abortRebase(ctx, t.path)`. There is no check whether the loop initiated the rebase. A repo already mid-conflict-resolution before `camp pull all` runs fails the pull because of that rebase, and the handler destroys the user's resolution progress. Single-repo `camp pull` correctly only prints guidance (`pull.go:77-84`).
- **Recommendation:** probe rebase-in-progress before pulling and skip with a warning; only abort a rebase the loop itself started.

## GIT-4. `camp status all` miscounts the first dirty file (MEDIUM)
- Verified this session. `internal/git/query.go:18` returns `strings.TrimSpace(output)`, stripping the leading status column of the first porcelain v1 line. `cmd/camp/status_all.go:216-230` reads `line[0]`/`line[1]`: a worktree-only modification ` M file` becomes `M file`, counted as Staged instead of Modified. Every table render and the JSON output is wrong whenever the first entry has a leading-space XY code.
- **Recommendation:** use `status --porcelain -z` with the correct parser that already exists at `internal/commands/workitem/staging_git.go:98-117`, or stop trimming porcelain output.

## GIT-5. `FilterTracked` silently drops non-ASCII paths (MEDIUM)
- `internal/git/commit.go:200-243` parses `git ls-files` without `-z`; with default `core.quotepath=true`, non-ASCII names come back C-quoted and never match raw input, so the path is excluded from commit scope. Callers: `cmd/camp/dungeon/move.go:241`, `cmd/camp/dungeon/crawl.go:338`, `cmd/camp/intent/crawl.go:178`. Sibling `ExpandTrackedPaths` (`commit.go:255`) uses `-z` correctly.
- **Recommendation:** add `-z` and NUL-split, matching ExpandTrackedPaths.

## GIT-6. `camp commit --amend` hazards (MEDIUM)
- `cmd/camp/commit.go:56` defaults `-a` true, so `camp commit --amend -m "fix typo"` first stages the entire tree, folding unrelated WIP into the amended commit. Separately, `--amend` with no `-m` skips the prompt (`commit.go:125`) and `executeCommit` builds `git commit --amend` with no `-m` via `CombinedOutput` with no tty (`internal/git/commit.go:47-68`), so git tries to spawn an editor headlessly and fails or hangs.
- **Recommendation:** do not auto-stage under `--amend` unless `-a` is explicit; add `--no-edit` for message-less amends.

## GIT-7. refs-sync silent skips and TOCTOU (MEDIUM)
- `cmd/camp/refs/commands.go:116-131`: `continue` on `ls-tree` failure (newly added submodule not in HEAD) or `rev-parse` failure (broken checkout); the user sees "All submodule refs are up to date" with those repos silently excluded. The staged-clean check at :58 happens before detection and staging at :94-103, so a concurrent stage slips into the unscoped commit.
- **Recommendation:** print skipped paths; scope the commit with `Only: toSync`.

## GIT-8. Worktrees misdetected as submodules wherever .git-is-a-file is the test (MEDIUM)
- `internal/git/submodule.go:17-30` (`IsSubmodule`) returns true for linked worktrees too. Consequences: `camp commit --sub` from inside a worktree reports "Operating on submodule" (`internal/git/resolve.go:90-98`); `projectsvc.ResolveFromCwd` falls into the accept-unknown-submodule branch (`internal/project/resolve.go:166-175`) and names the project `worktrees` via `nameFromPath` (`resolve.go:217-226`) with `LogicalPath: projects/worktrees`, which is what `--sync` would stage at root.
- **Recommendation:** distinguish by gitdir target (`.git/modules/` vs `.git/worktrees/`); `ResolveGitDir` (`internal/git/lock.go:89-127`) already reads it.

## GIT-9. Stale git index.lock removal trusts open-fd state (MEDIUM)
- `internal/git/lock_unix.go:33-123` decides staleness by "no process has the file open" (fuser/lsof). A live git process between creating index.lock and holding it open can have its lock deleted during retry cycles (`internal/git/retry.go:116`), allowing two index writers. The double-check (`lock_unix.go:162-172`) narrows but does not close the window; there is no lock-age floor. Errors fail safe (treated as active).
- **Recommendation:** require a minimum lock age in addition to the fd check.

## GIT-10. Plumbing duplication and small correctness debt (LOW)
- Confirmed plumbing variants: `internal/git/run.go:23` (`RunGitCmd`, unified classification, barely used), `internal/git/query.go:11` (`Output`, fmt.Errorf wrapping, stderr discarded), hand-rolled exec blocks in `internal/git/commit.go` (:67-84, :146-161, :183-192, :368-377), `internal/worktree/git.go` (own timeout wrapper and private gitError type), `pkg/commitkit/commitkit.go:169` raw exec for ShortHash, plus 51 non-test files calling `exec.CommandContext("git", ...)` directly. Converge on RunGitCmd plus a streaming variant.
- `internal/git/commit.go:140`: dead condition (`len(files) == 0 || files[0] != "--"` inside the else of the same check); the `"--"` sentinel contract with `StageAllExcluding` (:327) is implicit. Give the exclusion path its own function.
- `internal/git/commit.go:327-331`: exclusion pathspecs `":!"+p` are non-literal; a path with glob metacharacters becomes a wildcard. Use `":(exclude,literal)"`.
- Inconsistent lock paths: `run.go:56` hardcodes `index.lock` while `pull.go:126-131` resolves the real gitdir path.
- `internal/worktree/detect.go:125-128` joins HEAD onto a possibly relative gitdir without resolving against the worktree path (`clean.go:217-219` does it correctly).
- `internal/git/resolve.go:80-82` compares roots with `filepath.Abs` only, skipping EvalSymlinks (`internal/project/resolve.go:146-148` shows the right pattern).
- `internal/worktree/paths.go:57`: `strings.HasPrefix(absPath, wtRoot)` lacks a separator guard, matches sibling dirs like `worktrees-x`; benign today due to downstream validation.
- `cmd/camp/worktrees/commit.go:142` uses `PrependCampaignTag` only, so worktree commits never carry quest/workitem context unlike `camp commit` and `camp p commit`.
- Workitem-ref asymmetry: root commit auto-backfills missing refs (`cmd/camp/commit_workitem.go:30` EnsureRefForCommit), project commit does not (`cmd/camp/project/commit_workitem.go:18-26`), so old workitems silently lose the WI tag only on project commits.

---

# H. Intents, Workitems, Quests, Dungeon, Workflow/Flow/Concept

The three headline boundary claims in this section (WF-1, WF-2, WF-7) were independently re-verified by direct read in this session.

## Fest boundary violations (the two real ones found in the whole repo)

### WF-1. Campaign-root dungeon triage offers the entire fest-owned `festivals/` tree as a move candidate (HIGH)
- `internal/dungeon/service_list.go:125-136` (verified verbatim): the built-in `excluded` map contains `dungeon`, `.campaign`, `.git`, doc files, and dot-files, but NOT `festivals` or `projects`.
- The other exclusion layers do not fire in a real campaign: there is no `OBEY.md` inside `festivals/`, no root `.workflow.yaml`, and no default `dungeon/.crawl.yaml`.
- **Why it matters:** `camp dungeon crawl` or `camp dungeon move festivals archived` at the campaign root will `os.Rename` the whole fest-owned tree into `dungeon/archived/<date>/festivals` and auto-commit it. One keystroke relocates another tool's state directory.
- **Recommendation:** add `festivals` and `projects` to the built-in exclusions, and consider shipping a default `.crawl.yaml` at campaign init.

### WF-2. Dungeon resolver adopts fest-owned `festivals/dungeon/` as a camp dungeon context (HIGH)
- `internal/dungeon/resolver.go:64-72` (verified verbatim): walking up from cwd, the first directory containing a `dungeon/` subdirectory wins, with no check for fest ownership. fest's terminal-status directory is exactly `festivals/dungeon/`.
- Consequences when camp dungeon commands run inside `festivals/`: camp lists `planning/`, `active/`, `ready/`, `ritual/`, `chains/` as triage candidates; writes camp's dated buckets `festivals/dungeon/<status>/YYYY-MM-DD/` which diverge from fest's `dungeon/<status>/<festival>` layout; appends camp's `crawl.jsonl` into the fest-owned dir; and `camp dungeon add` scaffolds camp's `OBEY.md` and `.gitkeep`s there.
- **Recommendation:** the resolver should refuse or skip any `dungeon` directory under `festivals/`.

## Migrations and overwrite safety

### WF-3. Repair migration renames with no destination check; POSIX rename clobbers files silently (HIGH)
- `internal/scaffold/repair.go:767`: `os.Rename(src, dst)` inside `ExecuteMigrations` with no stat of dst. For files (unlike directories) rename silently replaces an existing destination. Re-running `camp init --repair` after a partial failure, with a same-named item already in the dated bucket, destroys data with no error. Compounds with INIT-1 (partial migration auto-committed).
- **Recommendation:** use the existing `statuspath.ExistingItemPath` check and fail with ErrAlreadyExists; pre-validate the whole plan.

### WF-4. `MigrateV1ToV2` renames onto the workflow root with no collision check (HIGH)
- `internal/workflow/service_sync_migrate.go:184-187`: `os.Rename(src, dst)` after `dst := filepath.Join(s.root, name)` for everything in `active/` and `ready/`. If `active/foo.md` and root `foo.md` both exist, or active and ready share a name, the second rename silently overwrites the first. Partial failure aborts mid-loop with the schema still v1.
- **Recommendation:** apply the destination-exists check `Move` already uses (`internal/workflow/service_move.go:46-50`); collect per-item errors.

### WF-5. `CreateWithEditor` can silently overwrite an existing intent (HIGH)
- `internal/intent/service.go:199-206` plus `internal/intent/helpers.go:46-49`: the editor path does `moveFile(tmpPath, finalPath)` where moveFile starts with bare `os.Rename`. `CreateDirect` stat-guards (`service.go:115-117`); the editor path does not. If the user edits the `id:` to an existing id, the existing intent is destroyed with no error.
- **Recommendation:** the same ErrFileExists guard, ideally via `O_CREATE|O_EXCL`, before moveFile.

### WF-6. Atomic-write discipline is split down the middle (HIGH, highest-leverage fix in this section)
- `fsutil.WriteFileAtomically` exists, is correct (same-dir temp, fsync, rename), and is used by config, links, the priority store, pins, and the nav cache. But these all truncate in place with plain `os.WriteFile`:
  - intent: `internal/intent/service.go:120,292,354`, `internal/intent/rename.go:49,57`, `internal/intent/notes.go:111,162` (Save rewrites the only copy in place; crash mid-write is unrecoverable)
  - quest: `internal/quest/quest.go:271`
  - workflow schema: `internal/workflow/service_init.go:58`, `service_sync_migrate.go:101,221`
  - flow registry: `internal/flow/registry.go:76`
  - and `internal/commands/workitem/create.go:249-259` carries a weaker private `atomicWriteFile` clone (fixed .tmp name, no fsync) one import away from the real helper, which `doctor_ref_backfill.go:149` already uses for the same file type.
- **Recommendation:** one sweep routing all of these through `fsutil.WriteFileAtomically`; delete the private clone.

## Boundary-adjacent

### WF-7. Intent promote writes directly into fest-internal festival layout (MEDIUM)
- `internal/intent/promote/promote.go:344-359` (verified verbatim): `copyIntentToIngest` hardcodes `festivals/<dest>/<festival>/001_INGEST/input_specs` and even `MkdirAll`s it when fest did not create it. Festival creation itself correctly shells out to the fest CLI (`promote.go:293-312`) and no fest.yaml or festival state is touched.
- **Why it matters:** if fest changes its ingest layout, camp silently writes orphan files. This is the documented "consumer reaches behind the tool" anti-pattern.
- **Recommendation:** capture a `fest ingest <file>` capability as a fest workitem and shell out; until then do not MkdirAll fest-internal directories (skip the copy when the dir is absent). Also: fest CLI stderr is discarded on promote failure (`promote.go:309-312` returns a bare Result); surface `exec.ExitError.Stderr`.

## Intents (internal/intent)

Verified fixed in HEAD: the two recent collision-safety commits are real. `Convert` stat-checks the target (`internal/intent/notes.go:154-156`); Move/UpdateDirect/Edit route through `moveTargetPath` (`service.go:437-441`) preserving renamed slugs; `collisionSafeMovePath` (`service.go:447-458`) plus `pathAvailableForID` (`service.go:463-469`) check the occupant's frontmatter id and suffix `-2, -3, ...`. Regression tests exist (`rename_test.go:252,264,277`, `service_test.go:1876`).

Remaining:
- MEDIUM: rewrites drop unknown frontmatter keys and comments. `internal/intent/parse.go:22-42` (fixed schema) plus `SerializeIntent` (`parse.go:150-174`) marshal only known fields; any user-added key is stripped on the next Move/Rename/Edit. Round-trip unknown keys via yaml.Node or document the supported-key contract.
- MEDIUM: frontmatter delimiter matching is substring-based, not line-anchored (`parse.go:104`: `bytes.SplitN(content, delimiter, 3)`). A `---` inside a value splits mid-line and the corruption persists on rewrite; content before the first delimiter is dropped. Anchor delimiters to lines.
- MEDIUM: TOCTOU on every guarded path; collision checks are stat-then-WriteFile (`service.go:115-120,341-354`, `notes.go:154-162`, `rename.go:102-111`). festival-app shells out to camp, so concurrent processes are real. Use `O_CREATE|O_EXCL` and retry the suffix loop on EEXIST.
- MEDIUM: `removeAllCopies` deletes by filename without verifying frontmatter id (`service.go:474-482`); a different intent whose renamed basename matches gets deleted.
- MEDIUM: gather swallows archive failures (`internal/intent/gather/gather.go:217-228`, `continue` with a comment claiming logging that does not exist). Collect per-source errors into GatherResult.
- LOW: `generateMergedID` produces a format-reversed id (`merge.go:65-69` vs canonical `slug.go:71-78`); latent today, but MergeIntents is exported. LOW: stale id-index in long-lived processes (`idindex.go:36-38`, no re-stat). LOW: same-second duplicate capture errors instead of suffixing (`slug.go:71-78`). LOW: `migrateLegacyDir` removes source when same-named destination exists, comparing filename only (`migration.go:368-370,384-387`). LOW: `cmd/camp/intent/promote.go:91` and `move.go:91` report any Find error as "intent not found", discarding the cause.

## Workitems (internal/workitem, internal/commands/workitem)

Fest boundary clean: all writes go to `.campaign/settings/workitems.json`, `.campaign/workitems/{links,current}.yaml`, and `workflow/<type>/<slug>/.workitem`; festival discovery only reads `fest.yaml`.

`camp workitem priority` (PR #318), reviewed directly: good shape overall. Validation-first ordering (`priority.go:69-72`), typed validation errors, case-insensitive aliases (`med`, `none`), clear path deletes the store file when empty, contract-versioned JSON, strong table-driven tests (invalid level leaves no store file; cancelled context mutates nothing). Findings:

- MEDIUM: priority store has no file lock; load-modify-save races lose writes. `internal/commands/workitem/priority.go:85-97`: `priority.Load(storePath)` then `priority.SaveOrDelete(...)`; same unlocked pattern in the TUI (`internal/workitem/tui/update.go:296-297,318-319`). The repo already solved this for links with `links.WithLock` over `fsutil.AcquireFileLock` (`internal/workitem/links/save.go:72-97`); the priority store is the one mutable contract file that skipped it. Add `priority.WithLock`.
- MEDIUM: `camp workitem` list (including `--json`) writes to disk as a side effect. `internal/commands/workitem/workitem.go:87-91`: `if priority.Prune(store, validKeys) { priority.SaveOrDelete(...) }`. A read command mutates and can delete `.campaign/settings/workitems.json`; a transient discovery gap (festival mid-move between stage dirs) permanently destroys that item's priority and dirties git during a read. Move pruning to explicit write paths (priority command, TUI mutations, doctor --fix).
- MEDIUM: stage values are magic strings duplicated in four drifting places: `internal/commands/workitem/workitem.go:230` (validStages), `internal/workitem/discover_intents.go:21`, `internal/workitem/discover_festivals.go:34`, `internal/commands/workitem/staging_festival.go:53`.
- MEDIUM: stage filter is not context-aware: `internal/workitem/discover_workflow_docs.go:77` sets `LifecycleStage: ""` for design/explore/custom items and `filter.go:21` can never select empty, so `--stage` silently hides them; the valid set conflates per-type vocabularies (inbox only matches intents, planning only festivals). This is the camp-side root of the known festival-app stage-chips issue. Publish per-type stage vocabulary in the JSON payload and represent "no stage" explicitly.
- MEDIUM: `doctor --fix` swallows the post-fix rescan error (`internal/commands/workitem/doctor.go:118-119`: `knownIDs, _ = ...`); a failed rescan reports every link broken, exit 2.
- LOW: vacuous integration tests (`internal/workitem/priority/integration_test.go:101-103` slices a local and asserts an untouched store; the filter-safety test at :63-85 is equally tautological). LOW: JSON `sort.direction: "desc"` misdescribes the composite sort (`internal/workitem/json.go:69-73`). LOW: priority rank table duplicated (`internal/workitem/sort.go:31-42` vs `priority/priority.go:18-29`, agree today, nothing ties them; add a cross-package agreement test). LOW: festivals in `ritual/` and `chains/` invisible to discovery (`discover_festivals.go:34`) but recognized by commit staging (`staging_festival.go:52-54`); priorities keyed to them get pruned by the list side effect. LOW: `workitem link --festival <id>` hardcodes `festivals/active/` (`internal/commands/workitem/link.go:222-226`). LOW: `fmt.Errorf` at `workitem.go:63,108,203,227,233,237,240,243,257`. LOW: `counts` ignores custom types (`json.go:46-58`) so total != sum(by type).

## Quests (internal/quest, cmd/camp/quest, dev channel)

Dev gating verified clean: all of `cmd/camp/quest/*.go`, `cmd/camp/quest_register.go`, `internal/quest/tui/*` carry the dev tag; `release_profile_test.go:15` pins it; the untagged `internal/quest` library is intentional since stable code references it. Quest autocommit is selective (`git commit --only`), never sweeps unrelated staged changes. Fest boundary clean.

- MEDIUM: quest UX leaks into stable builds while the feature is invisible there: `internal/commands/workitem/create.go:46` and `adopt.go:31` register `--quest` with help text advertising quests; `ref_quest.go` resolves it; `resolve.go:91-97` prints `quest:`; stable `camp init` scaffolds the quest dungeon (`internal/scaffold/init.go:219`). Half in, half out (cross-ref BR-5). Document as forward-compat or gate the flag.
- MEDIUM: `List` hard-fails on any single bad entry, bricking every quest command: `internal/quest/quest.go:134-137` returns on the first Load error; one corrupt quest.yaml kills list/show/link/lifecycle and CAMP_QUEST resolution. Skip-and-warn.
- MEDIUM: non-atomic quest writes (`internal/quest/quest.go:271`, cross-ref WF-6); combined with the hard-fail List, one crash mid-write bricks the subsystem.
- MEDIUM: TOCTOU slug reservation (`internal/quest/service.go:480-483` reserves by os.Stat; creation later via MkdirAll which succeeds on existing dirs); two concurrent same-day same-name creates silently overwrite each other's quest.yaml. Claim with os.Mkdir, treat EEXIST as try-next-suffix.
- LOW: quest ID uniqueness never checked at create (`internal/quest/slug.go:55-61`; Resolve returns first match). LOW: stdlib `errors.New` at `service.go:73,169,296,398`, `fmt.Errorf` at `cmd/camp/quest/lifecycle.go:128`. LOW: `service.go` is 593 lines with Create/CreateWithEditor duplicating ~5 steps (`service.go:67-104` vs `163-233`). LOW: `normalizeLinkPath` (`cmd/camp/quest/link.go:91-99`) treats relative input as campaign-root-relative regardless of cwd.

## Dungeon (internal/dungeon, cmd/camp/dungeon)

Collision policy is sound where it exists: every move op errors on an existing destination, and `statuspath.ExistingItemPath` scans legacy flat plus all dated buckets. Beyond WF-1/WF-2:

- MEDIUM: stat-then-rename TOCTOU on every move (`internal/dungeon/service_move.go:37-43`, repeated at :86-98, :132-144, :183-195 and `docs_routing.go:125-131`); a destination appearing between Stat and Rename is silently replaced.
- MEDIUM: stat error conflation defeats the guard: `service_move.go:29-31` maps any source stat error to ErrNotFound, and the destination check at :37 triggers only on err == nil, so a permission error against the target falls through to rename-anyway. Distinguish os.IsNotExist like `MoveToStatus` does at :69.
- MEDIUM: root-directory model contradiction: `MoveToDungeon` moves directories to the dungeon root, but `ListItems` skips all root directories as status dirs (`service_list.go:76-79`) while `ListStatusDirs` (`service_list.go:33-50`) treats every root subdirectory as a status. A directory moved to the root becomes invisible to list and appears as a bogus destination in the status picker.
- MEDIUM: no collision tests for exactly the two checks whose failure is silent overwrite (`MoveToDungeon` at `service_move.go:37`, `MoveToDocs` at `docs_routing.go:125`); MoveToStatus collisions are well tested (`service_move_test.go:165-205,252-292`).
- MEDIUM: `docs_browser.go` is 719 lines, `Update` 111 lines (:227-337), `View` 104 (:544-647); strictly read-only so no data risk; split fs/model/view.
- LOW: `MoveToDungeon` lacks the `pathutil.ValidateBoundary` check its siblings have (`service_move.go:16-46` vs :121 and `docs_routing.go:66`). LOW: `Archive`/`ArchivedPath` have no production callers and a weaker ad hoc validation (`service_move.go:155-162`). LOW: `ListStatusDirs` silently drops unreadable status dirs (`service_list.go:40-43`). LOW: wrong hint at `cmd/camp/dungeon/crawl.go:36` ("camp dungeon init"; the command is `camp dungeon add`); triage warnings print to stdout (`triage.go:95,104,112`) while list correctly uses stderr. LOW: `"archived"` re-hardcoded at `service.go:65`, `service_move.go:162`; `"dungeon"` literal at `resolver.go:65`, `service_list.go:126`, `workitem_move.go:195`.

## Workflow / flow / concept

What they are: `internal/workflow` is a generic schema-driven status-directory state machine (`.workflow.yaml`). `internal/flow` is two unrelated things fused under one verb: a shell-command registry/runner (`.campaign/flows/registry.yaml`, `camp flow run/list`) and the CLI front-end for the workflow package (`camp flow add/move/items/status/sync/migrate`). `internal/concept` is read-only navigation metadata. (Cross-ref ARCH-3 for the naming inversion.)

- HIGH (architecture): this repo contains at least four parallel implementations of "move a markdown item between status directories", with divergent collision policies: `internal/workflow/service_move.go:44-59` (check then rename), `internal/dungeon/service_move.go` (four near-identical check-then-rename blocks), `internal/intent/helpers.go:46-50` (rename with cross-device fallback, no check at call sites that need one), and two movers with no check at all (`internal/scaffold/repair.go:767`, `internal/workflow/service_sync_migrate.go:184-187`, both silently overwrite). The `dungeon/{completed,archived,someday}` taxonomy is independently hardcoded in `internal/workflow/defaults.go:40-53,100-113`, `internal/commands/workflow/create.go:20-24`, `internal/dungeon/scaffold`, and intent. **Recommendation:** extract one shared statusdir move primitive (collision policy, dated buckets, cross-device fallback, no-replace semantics) and decide whether schema-driven workflow supersedes hardcoded dungeon.
- Commit 1f42303 concept migration verified SAFE: `migrateConceptsToNested` (`internal/scaffold/repair.go:352-404`) rewrites only the in-memory concept list in campaign.yaml via `fsutil.WriteFileAtomically`, never moves user files, runs only behind the repair confirmation, and is idempotent with tests. Minor: `conceptListsEqual` (`repair.go:437-447`) compares only Name/Path/len(Children), so a child rename previews as "no change" while still being written.
- MEDIUM: flow runner argument splicing is shell injection via args: `internal/flow/runner.go:32-37`: `command = command + " " + strings.Join(extraArgs, " ")` into `sh -c`. The registry command being shell is the trust model (like just); the arg splicing is the bug: `camp flow run deploy -- "us east"` word-splits, and `;` or `$(...)` in an arg executes. Pass args as positionals: `sh -c '<cmd> "$@"' sh args...`. (Same family as RUN-1.)
- MEDIUM: history tracking advertised but unimplemented: `internal/workflow/service_move.go:87-91` is a TODO stub returning nil, while `TrackHistory: true` is the schema default, `camp flow show` prints "History: enabled", the long help lists an unregistered `camp flow history` subcommand, and the call site swallows the error (`service_move.go:79-81`). Implement the JSONL append (dungeon's AppendCrawlLog is the pattern) or stop advertising it.
- MEDIUM: `camp flow migrate` help contradicts behavior: `internal/commands/flow/migrate.go:30-31` claims ready/ items move to `dungeon/ready/` but the code moves them to the root; :35 advertises `--force to skip confirmation` but no confirmation exists.
- MEDIUM: `internal/concept/service.go` is 680 lines because the dead `FSService` (:385-681) is a near-verbatim copy of `DefaultService` (:23-382) with `os.ReadDir` swapped for `fs.ReadDir`. Collapse to one implementation over an injected fs.FS (or just delete the dead copy per ARCH-5).
- LOW: nested-concept depth disagreement (`internal/concept/children.go:20-35` maps one level; `cmd/camp/concepts.go:110-112` and `internal/commands/workflow/internal.go:53-62` recurse arbitrarily). LOW: `context.Background()` instead of `cmd.Context()` in `internal/commands/flow/sync.go:29`, `show.go:33`, `items.go:33`. LOW: dead/stub code: `DetectTransition`/`findItemStatus` (`internal/workflow/transition.go:26-85`), `camp flow show --tree` flag registered never read (`show.go:15,94`), `Service.Crawl` (`service_crawl.go:9-48`) fabricates "Reviewed/Kept" results while doing nothing, `show.go:65` ranges a map so output order is nondeterministic. LOW: `fmt.Errorf` throughout `internal/flow` (runner.go:27,45; registry.go:47,52,68,73,77), `internal/workflow/service_sync_migrate.go:167`, `internal/concept/service.go:329,369,636,676`.

## Cross-cutting observations for this section

- ID schemes: three formats, three collision strategies. Intents `slug-YYYYMMDD-HHMMSS` (seconds resolution, collide-then-error, `internal/intent/slug.go:71-78`); quests `qst_YYYYMMDD_xxxxxx` (crypto random, uniqueness never verified, `internal/quest/slug.go:55-61`); workitems `type-slug-YYYY-MM-DD` with a real collision scan and random hex suffix retry (`internal/commands/workitem/create.go:169-208`, the best of the three) plus derived `WI-<sha256/6>` refs with deterministic re-roll (`internal/workitem/ref.go`). Document the matrix; converge new code on the workitem pattern.
- Slug logic copy-pasted: `internal/intent/slug.go:30-65` and `internal/quest/slug.go:21-42` are the same algorithm with renamed regex vars. Extract a shared slug package before a third copy appears.
- Frontmatter parsing: at least three independent implementations (`internal/intent/parse.go` substring-split, `internal/intent/feedback/{scanner,builder}.go` prefix-checked, `internal/workitem/metadata.go` whole-file YAML). The intent one has the line-anchoring bug; consolidating fixes it once.
- Status/dir constants: intent is the model citizen (typed Status constants doubling as relative dirs, `internal/intent/types.go:23-59`); workitem stages, dungeon statuses, and workflow statuses are magic strings duplicated across files (cross-ref ERR-7).

## What works well in this section

Intent status moves are genuinely collision-safe now (both recent fix commits verified in HEAD with regression tests). Workitem JSON contract discipline is strong. The camp/fest write boundary is respected everywhere except the two dungeon footguns and the promote ingest copy. Quest lifecycle transitions are validated rather than any-to-any; quest autocommit is correctly selective. The concept-nesting migration from 1f42303 is idempotent and atomic. camperrors usage and context propagation are near-universal outside the listed stragglers.
