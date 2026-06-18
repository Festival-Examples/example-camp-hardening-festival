# Festival Purpose: camp-hardening (CH0001)

## End Goal

Remediate every finding from the fable-review-2026-06-10 staff-level review of `projects/camp` (recommendations R1-R36 plus the second-pass additions N-1 through N-27), in dependency order: the data-loss class first, then gate enforcement, fest-boundary violations, the atomic-write and locking sweep, agent-facing contract bugs, and finally the low-severity cleanup sweeps.

## Why It Matters

camp is the campaign CLI that every agent loop and the festival-app shell out to daily. The review found:

1. **Destructive code paths that can destroy user data in routine operation.** `camp worktrees clean` rm-rfs dirty trees with no prompt; `camp cp --force` truncates source files to zero when src and dest resolve to the same path; `camp fresh` force-removes dirty worktrees of merged branches; `camp project remove` deletes `.git/modules` history holding unpushed branches; the `camp sync` stale-ref fallback RemoveAlls non-empty submodule dirs and replaces them with structurally wrong clones. These sit directly under daily festival operation.
2. **A red test gate on main with no enforcement.** The manifest test fails in both profiles, dev-tagged tests never run in any gate, five integration-tagged host test files are dead (one does not compile), and no local enforcement exists to stop regressions from merging.
3. **Fest-boundary violations.** camp's dungeon triage offers the fest-owned `festivals/` tree as a move candidate, and the dungeon resolver adopts `festivals/dungeon/` as a camp dungeon context. One keystroke can relocate another tool's state directory.
4. **Half-adopted durability primitives.** `fsutil.WriteFileAtomically` and `AcquireFileLock` exist and are correct, but intent, quest, workflow, flow, state, and config-allowlist writes truncate in place, and the registry read-modify-write cycle is unlocked while concurrent camp processes are the norm in agent loops.
5. **Agent-facing contract bugs.** festival-app bundles dev-profile camp and parses its JSON: `flow items --json` never emits JSON, `doctor --json` exits 0 on failure, the intent JSON contract is unversioned, the stage filter silently hides design/explore items, `workitem create` is non-atomic in exactly the shape that breaks the app's FA0011 retry model, and no surface reports the build profile.
6. **Standards debt** that grows with every PR because nothing enforces it: 327 production `fmt.Errorf` occurrences, ~216 dead symbols (~2,000+ lines), 14 files over the 500-line standard, two parallel command trees, four parallel statusdir-move implementations.

## Success Criteria ("done")

The festival is complete when all of the following hold on the camp-hardening branch (linked worktree `projects/worktrees/camp/camp-hardening`):

### Functional

- Data-loss class eliminated and regression-tested: worktrees clean (R2 plus second-pass correction 1), cp same-file/self-recursion guards (N-1), fresh/prune dirty-worktree skip (N-2, R16), project remove unpushed-ref guard (N-3), sync fallback consent plus real submodule machinery (N-6) with checkout errors surfaced and validateUpdate corrected (N-10), all-or-nothing migrations with no auto-commit of failures (R4: INIT-1, WF-3, WF-4), moveFile ErrFileExists guard with EXDEV-gated fallback (R10, N-19), promote rollback removes only what it created (N-12).
- Test gate green: manifest allowlist includes `workitem priority`; `go test ./...` passes with and without `-tags dev` (R1).
- The five dead integration-tagged host test files migrated into `tests/integration/` or deleted; `go vet -tags=integration ./...` runs in `just lint` (R13, R14); container pool Reset covers `/root/.obey` and per-test work roots (R15).
- Local gate enforcement for both profiles: `just test` covers dev-tagged tests, release recipes gate on both profiles, and a pre-push hook runs build, test, and lint locally. No GitHub Actions workflows added or modified.
- Fest-boundary regression tests: `festivals` and `projects` in the built-in dungeon exclusion map; resolver skips any `dungeon/` whose parent chain includes `festivals/` (R3: WF-1, WF-2).
- Zero `os.WriteFile` call sites on the intent, quest, workflow, flow, state, gitignore-append, workitem private-clone, and config-allowlist paths; everything routes through `fsutil.WriteFileAtomically` (R11: WF-6, N-21).
- Registry and priority-store read-modify-write cycles run under `fsutil.AcquireFileLock` with re-load inside the lock; the fsutil stale-lock TOCTOU is fixed (R12: REG-1).
- `cr`/`camp run` project just-dispatch execs `just` with argv (no `sh -c` flattening); flow runner passes extra args as positionals; tested with spaces, quotes, and metacharacters (R5: RUN-1).
- Scoped-commit semantics fixed in order: N-4 (`--only` worktree-content over-commit), N-5 (staged-rename corruption), then R6 (syncParentRef, SyncSubmoduleRef, refs-sync scoping), N-14 (root commit final-commit scoping); the fest-side mirror of the commitkit scoping captured as a fest workitem, not done in camp.
- `camp pull all` never aborts a rebase it did not start (R7); `camp doctor --json` exits non-zero on failing checks (R8).
- Reads do not write: workitem list prune side effect (R18), intent EnsureDirectories migration on read (N-8), status-all cache write (N-18) all moved to explicit write paths.
- Agent/app contract fixes: `flow items --json` emits versioned JSON with errors to stderr and non-zero exit (N-7); `workitem create` atomic or self-cleaning (N-9); intent JSON contract versioned (R20); stage filter no longer hides design/explore items and the per-type stage vocabulary is published (R26); `intent list --type` honors all values (N-15); path semantics unified to campaign-relative plus `campaign_root` (N-16); manifest annotates every shipped command (N-11); version stamping plus a `Profile` field in `camp version --json` (R21, N-25).
- README install path corrected and the flow channel contradiction decided and fixed (R9); `reg` alias collision removed and `SilenceUsage` set on root (R17).
- Cleanup sweeps complete: R19 (JSON flags and error envelope), R22 (exit-code table), R23 (signal-aware root context), R24 (fmt.Errorf forbidigo ratchet), R25 (shared statusdir move primitive), R26, R27 (shell wrapper and nav), R28 (repair quiet failures and TTY guards), R29 (dead code, strictly before R30), R30 (cmd-logic extraction), R31 (git plumbing convergence), R32 (quest robustness), R33 (intent parser hardening), R34 (flaky-test hygiene), R35 (help and docs polish), R36 (misc small fixes), plus the remaining N findings (N-13, N-17, N-20, N-22, N-23, N-24, N-26, N-27).

### Quality

- `just build`, `just test`, and `just lint` pass in BOTH profiles (stable and `BUILD_TAGS=dev`) at every implementation gate, verified by running both.
- Every behavioral fix in the data-loss, fest-boundary, and contract classes ships with a regression test; destructive-path fixes get container integration tests.
- Zero `os.WriteFile` on the R11/N-21 state paths, verified by grep at the final gate.
- `go vet -tags=integration ./...` is part of `just lint` and passes.
- No new GitHub Actions workflows exist in the diff.

### Completion

- All P0/P1 findings (R1-R18 plus N-1, N-2, N-3, N-6) fixed with regression tests; all P2/P3 items (R19-R36 plus remaining N findings) fixed or explicitly dispositioned in the decision log.
- `go test ./...` passes with and without `-tags dev`; `go vet -tags=integration ./...` clean.
- Local gate enforcement (both-profile just recipes plus pre-push hook) in place with no GitHub Actions changes.

## Traceability

- Source corpus: `001_INGEST/input_specs/` (review-README.md, recommendations.md R1-R36, second-pass-fable.md N-1 to N-27, seven findings docs).
- Festival intent: `FESTIVAL_GOAL.md` (treated as authoritative).
