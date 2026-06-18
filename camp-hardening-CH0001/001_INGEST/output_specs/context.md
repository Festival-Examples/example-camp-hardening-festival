# Context: camp-hardening (CH0001)

## The review corpus (source of truth)

All in `001_INGEST/input_specs/`:

| File | What it is |
|---|---|
| `review-README.md` | First-pass review overview: methodology, severity summary (3 critical, ~24 high, ~45 medium, ~60 low), what is in good shape, doc index |
| `recommendations.md` | R1-R36 prioritized with severity, file:line locations, approach, and step counts; suggested four-phase festival shape |
| `second-pass-fable.md` | Adversarial re-verification (20/20 confirmed, 0 refuted, GIT-1 and WF-5 WORSE than reported), three precision corrections, fresh-hunt findings N-1 to N-27, required reordering (section 4), and the hunted-and-found-clean list |
| `findings-architecture.md` | ARCH-1..10: two command trees, logic-in-cmd, dead code inventory (~216 symbols), file-size violations, import hygiene, dependency pins |
| `findings-command-ux.md` | UX-1..22: alias collisions, dead global flags, JSON flag split, exit codes, TTY guards, stream discipline |
| `findings-subsystems.md` | REG/INIT/LINK/NAV/RUN/SH/GIT/WF findings with verified mechanics for registry, init/repair, intake, navigation, camp run, shell, git/commit, intents/workitems/quests/dungeon/workflow |
| `findings-error-handling-standards.md` | ERR-1..9: 327 fmt.Errorf occurrences, no signal context, swallowed errors, magic strings, locale-fragile classification |
| `findings-testing.md` | TEST-1..9: red gate, dead tagged tests, container Reset gap, zero-test packages ranked by danger, isolation reality, flakiness |
| `findings-build-release.md` | BR-1..10: untested dev tags, ungated releases, unstamped binaries, invisible channel, flow contradiction, broken README install |
| `findings-json-contracts.md` | JSON-1..6: jsoncontract scope, the workitem v1alpha5 model contract, intent contract gap, omitempty hazards, festival-app stability matrix |

Review baselines: first pass at camp `5c22d2b` (2026-06-10), second pass at `11c270f` (2026-06-11). Both passes were documentation-only.

## The codebase

- `projects/camp`: Go CLI, 917 files, ~186k lines, cobra, ~160 commands in 9 help groups. Module main at `./cmd/camp`.
- Implementation worktree for this festival: `projects/worktrees/camp/camp-hardening` (linked project; read-only during planning).
- Build channels: dev/rc/stable via `//go:build dev`; `CLI_PROFILE=dev` -> `BUILD_TAGS=dev`. `quest` and `flow` are dev-only commands today.
- Strong spots the review certified (build on, do not rewrite): the workitem JSON contract discipline (v1alpha5 family with changelog), the `internal/git/commit` scoped-commit engine (modulo N-4/N-5), the container test harness (pooled testcontainers, 345 tests), `internal/fsutil` atomic write and lock primitives, near-universal context propagation into git execs, `tools/release-notes`.

## Downstream consumers and coordination surfaces

- **festival-app** bundles DEV-profile camp and fest binaries and shells out to them, parsing JSON. It depends on: `workitem ... --json` (model contract), `workitem priority --json` (PR #318), `doctor --json`, `flow items --json` (currently broken, N-7), intent JSON (unversioned, R20), the `__manifest` agent-gating surface (incomplete, N-11), `camp version --json` (no profile field, R21/BR-4). Schema bumps must be coordinated, versioned changes.
- **fest** imports `pkg/commitkit` (camp's public facade). The R6 scoping change to `SyncSubmoduleRef` is a fest-facing contract change; fest also independently reimplements the unscoped pattern (`projects/fest/internal/commands/commit/commit.go:541-553`), captured as a fest workitem rather than fixed here.
- **FA0011 precedent:** festival-app's create-festival retry was made chain-only, never re-create, after fighting partial-success states. camp's `workitem create` non-atomicity (N-9) makes that retry model unsatisfiable; the fix restores compatibility.
- **guild-scaffold and obey-shared** are pinned at pseudo-versions (ARCH-9); tagging them is part of R36.

## Prior art and patterns to copy (in-repo)

| Need | Pattern |
|---|---|
| Locked read-modify-write | `internal/workitem/links/save.go:72-97` (`links.WithLock` over `fsutil.AcquireFileLock`) |
| Atomic file write | `internal/fsutil/atomic_write.go` (same-dir temp, chmod, fsync, rename) |
| Correct porcelain parsing | `internal/commands/workitem/staging_git.go:98-117` (`-z` parser, rename-aware) |
| Collision-checked status move | `internal/workflow/service_move.go:44-59`; `statuspath.ExistingItemPath` for dated buckets |
| JSON error envelope | `internal/jsoncontract` (`RunE`, `Requested`); consumed by workitem/workflow families |
| Versioned JSON schema | `internal/workitem/json.go` (workitems/v1alpha5 with documented changelog) |
| Dirty-worktree check | `detachedWorktreeClean` at `internal/prune/prune.go:302-323` (exists for detached, missing for branch worktrees) |
| Submodule-add failure cleanup | `cleanupFailedSubmoduleAdd` at `internal/project/add.go:243-267` |
| TTY degradation | `cmd/camp/intent/add.go:162-164`, `cmd/camp/switch.go:86-88` |
| Lint ratchet with allowlist | `lint-no-host-fs-tests` in the justfile (model for the fmt.Errorf forbidigo ratchet) |
| Refs-sync staged-clean precheck | `cmd/camp/refs/commands.go:57-62` |

## Known environmental gotchas

- macOS `/var` vs `/private/var`: every path comparison needs `EvalSymlinks` (pattern in `internal/pathutil`); intent JSON already emits symlink-canonicalized absolute paths that fail prefix-matching against unresolved roots (N-16).
- POSIX `os.Rename` silently replaces existing FILES (not non-empty dirs); this is the mechanism behind WF-3/WF-4 and why pre-validation matters.
- `git commit --only -- <path>` commits WORKTREE content, not the staged blob (N-4, empirically verified in scratch repos); any "scoped commit" reasoning must account for this until REQ-19 lands.
- The container pool resets `/root/.config/camp` (pre-migration path) but not `/root/.obey` (current registry home); tests inherit dirty containers scheduling-dependently until R15.
- `-short` is nearly inert in the suite (only 5 files check `testing.Short()`); per-package unit timeout is 120s.

## History that explains the present

- The red gate: `workitem priority` (camp PR #318) merged without any gate running because no enforcement exists and dev-tagged tests are never run by `just test`. This is the proof case motivating REQ-15.
- `commit_integration_test.go` has not compiled since ~2026-03-13 (commit 752f0b7) when submodule sync was removed from `camp p commit`; nothing vets `-tags=integration`.
- Intent move collision safety was recently fixed (two commits verified in HEAD with regression tests); the editor paths (WF-5/N-19) were missed by those fixes.
- The 2026-05-29 test-performance exploration (`workflow/explore/camp-test-suite-performance-2026-05-29/`) drove the container pool and parallel unit packages; its remaining open items (R3/R4/R6/R7 of that doc) are background, not festival scope except where TEST-6/TEST-8 overlap.
- GitHub Actions is intentionally paused campaign-wide; the second pass explicitly reframed R13's CI ask as local-only enforcement (section 4.7).

## Festival shape suggested by the corpus

The recommendations doc proposes four implementation phases, amended by the second pass:

1. **Stop the bleeding (P0):** R1-R10 plus promoted N-1, N-2, N-3, N-6 (and N-10 riding N-6); R2 widened by correction 1; N-2 merged into R16's test work; N-6 sharing R7's hostile-remote fixture.
2. **Durability and gates (P1):** R11 (+N-21), R12 first; R13-R15 unlock trustworthy gates; R16-R18 (+N-8, N-18); R5; the N-4 -> N-5 -> R6 -> N-14 commit-semantics chain.
3. **Contracts and consistency (P2):** R19-R28 with the festival-app cluster (N-7, N-9, R20, R26, N-15, N-16, N-11, R21/N-25, N-17, N-24) prioritized.
4. **Debt and polish (P3):** R29 strictly before R30, then R31-R36 and the remaining N items (N-13, N-20, N-22, N-23, N-26, N-27).

The 002_PLAN phase decides the final phase/sequence breakdown; this ordering is the constraint set it must respect, not necessarily the literal structure.
