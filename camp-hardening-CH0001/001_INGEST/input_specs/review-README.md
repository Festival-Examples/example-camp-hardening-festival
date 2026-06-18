# camp Staff-Level Code Review

**Date:** 2026-06-10
**Project:** `projects/camp` at commit `5c22d2b` (branch main)
**Scope:** the entire repo including `tools/`: architecture, command UX, every core subsystem, error handling and standards compliance, concurrency and filesystem safety, test suite, build channels and release pipeline, docs accuracy, and the JSON contracts festival-app depends on.
**Nature:** documentation-only review. No code, config, tests, or docs in the project were modified. These findings seed a remediation festival.

## Methodology

- Direct code reads plus standards greps by the lead reviewer, with seven parallel deep-dive review passes (architecture, command UX, registry/nav/run, git/commit, intents/workitems/quests/dungeon, testing, build/release/docs/JSON).
- Every critical and high finding was verified against the actual code at the cited file:line, most by two independent reads. Load-bearing claims were re-verified by the lead in this session, including: the red manifest test (run live), `doctor --json` exit path, the `reg` alias collision, `worktrees clean` fall-through RemoveAll, the `sh -c` argv flattening, registry lock absence, porcelain TrimSpace, the pull-all rebase abort, SyncSubmoduleRef commit scoping (one sub-claim corrected during verification), repair migration auto-commit, the dungeon festivals exclusion gap, the dungeon resolver fest adoption, the promote ingest write, the README install path, and the dead justfile ldflags variable.
- Live-binary checks were used where behavior claims needed them (`camp reg`, usage-dump on runtime error, `camp status --help`).
- Claims from review passes that did not survive verification were corrected or dropped (examples: a claimed zsh completion code-execution vector downgraded to latent escaping hygiene; a push.go copy-paste claim refuted; SyncSubmoduleRef staging scope corrected to staged-scoped but commit-unscoped).
- Context facts honored, not re-litigated: dev/rc/stable channel gating is intentional (quest dev-only on purpose); registry path migration is complete (no compat shims recommended); camp owns campaigns and fest owns festivals (boundary violations WERE flagged); host-vs-container test isolation reported factually against the 2026-05-29 exploration.

## Headline assessment

camp is a large (917 Go files, ~186k lines), feature-rich CLI with genuinely strong spots: the workitem JSON contract discipline, the `internal/git/commit` scoped-commit engine, the container test harness, atomic write and lock primitives in `internal/fsutil`, and near-universal context propagation into git execs. The dominant problems are unevenness and unguarded edges: two command trees and two error-handling generations coexist; the excellent fsutil primitives exist but half the subsystems do not use them; the test gate is red on main and no CI exists to catch it; a handful of destructive paths (worktrees clean, dungeon triage over `festivals/`, migration renames, pull-all rebase abort) can destroy user or fest-owned data.

## Severity summary

| Severity | Count | Examples |
|---|---|---|
| critical | 3 | red test gate on main (TEST-1); dead non-compiling integration-tagged tests (TEST-2); worktrees clean data loss (GIT-1) |
| high | ~24 | dungeon fest-boundary footguns (WF-1, WF-2); migration clobber plus auto-commit (WF-3, WF-4, INIT-1); cr argv flattening (RUN-1); ref-sync sweeps staged changes incl. fest-facing commitkit (GIT-2); pull-all rebase abort (GIT-3); doctor --json exit 0 (UX-11); registry lost updates (REG-1); atomic-write split (WF-6); dev-tagged code never tested (BR-1); broken README install (BR-7); flow channel contradiction in scaffolded skills (BR-6); intent editor overwrite (WF-5); container Reset stale paths (TEST-3); prune untested (TEST-4); reg alias collision (UX-1); dead global flags (UX-6); usage-dump on runtime errors (UX-12); 216 dead symbols (ARCH-5); logic-in-cmd (ARCH-2); two command trees (ARCH-1) |
| medium | ~45 | porcelain miscount (GIT-4); non-ASCII path drops (GIT-5); amend hazards (GIT-6); worktree-as-submodule misdetection (GIT-8); stale-lock races (REG-2, GIT-9); workitem list write side effect; priority store unlocked; stage filter hides items; quest List hard-fail; flow runner arg splicing; unimplemented history feature advertised; intent frontmatter parsing; JSON idiom split; exit-code collisions; no signal context; magic strings; 14 files over 500 lines; missing channel in version JSON; stale CLI reference |
| low | ~60 | catalogued per file |

## What is in good shape (explicitly)

- `internal/git/commit/commit.go` scoped `--only` commits with rename-safe `-z` parsing, and cycle-based lock retry.
- `internal/fsutil` atomic write (fsync plus rename) and file lock; `internal/workitem/links` uses them correctly.
- Workitem JSON: versioned schemas with changelog, non-null arrays, the new `priority` command is well built.
- Container integration harness: pooled, tagged, parallel, identity-isolated.
- Fest boundary: respected everywhere except the three flagged spots (dungeon exclusion list, dungeon resolver, promote ingest copy); festival scaffolding correctly delegates to `fest init`.
- Submodule-add failure cleanup; argv-array git execution (no shell) throughout intake; intent move collision safety (recent fixes verified present with regression tests); concept-nesting migration verified safe and idempotent.
- `tools/release-notes` is wired into every release recipe and reasonably tested.

## Index

| Doc | Contents |
|---|---|
| [findings-architecture.md](findings-architecture.md) | command trees, logic-in-cmd, package overlaps, dead code, file-size violations, import hygiene, dependencies |
| [findings-command-ux.md](findings-command-ux.md) | command inventory, alias collisions, flag consistency, help text, exit codes, error message quality, TTY guards, stream discipline |
| [findings-subsystems.md](findings-subsystems.md) | A registry/state, B init/repair, C intake/linking, D navigation, E camp run, F shell integration, G git/commit, H intents/workitems/quests/dungeon/workflow |
| [findings-error-handling-standards.md](findings-error-handling-standards.md) | fmt.Errorf sweep (327 occurrences), context/cancellation, swallowed errors, magic strings, naming |
| [findings-testing.md](findings-testing.md) | red gate, dead tagged tests, container Reset gap, zero-test packages, isolation reality vs the 2026-05-29 exploration, flakiness |
| [findings-build-release.md](findings-build-release.md) | channel gating consistency, untested dev tags, no CI, unstamped releases, tools/release-notes, docs accuracy, scaffolded skills |
| [findings-json-contracts.md](findings-json-contracts.md) | jsoncontract scope, workitem/priority schemas, intent contract gap, omitempty hazards, festival-app stability matrix |
| [recommendations.md](recommendations.md) | R1-R36 prioritized with severity, location, approach, and complexity in steps; suggested festival phasing |
