# D001: N-4 Fix Approach for `--only` Commit Semantics

**Status:** accepted
**Date:** 2026-06-12

## Context

N-4 (second-pass, critical-adjacent): `internal/git/commit/commit.go` `stageAndCommit` builds `Only: expandedScope`, and `internal/git/commit.go:62-65` turns that into `git commit --only -- <paths>`. `git commit --only` commits the WORKTREE content of the listed paths, not the staged blobs (empirically reproduced in scratch repos by the review). Consequences: `camp workitem commit --staged` folds unstaged worktree edits into the commit despite documenting index semantics, and every auto-commit has a window where a concurrent process's half-written in-scope file gets committed. R6 (sync-commit scoping) extends the `Only` pattern to more call sites, so this must be fixed first (second-pass correction 3).

## Options

### Option A: Temporary index snapshot (GIT_INDEX_FILE)
Build a throwaway index: `GIT_INDEX_FILE=<tmp> git read-tree HEAD`, then `git add -- <paths>` (or `git update-index` from the real index for --staged), commit from that index without `--only`.
- **Pros:** True snapshot semantics for every caller; closes the concurrent-writer window at the moment of `add`; `--staged` commits exactly the staged blobs (copy the real index instead of re-adding worktree content); no behavior change for the common clean case.
- **Cons:** Larger change to the engine; two semantic modes needed (worktree-snapshot for auto-commit flows, staged-blob for `--staged`); interacts with the lock-retry wrapper; more test surface.

### Option B: Divergence guard before `--only`
Keep `--only` but run `git diff --quiet -- <paths>` (index vs worktree) first and fail loudly on divergence.
- **Pros:** Small change; makes the silent over-commit impossible.
- **Cons:** Turns a routine state (user edits a file after staging it) into a hard error for `workitem commit --staged`; does not close the concurrency window between check and commit; every auto-commit flow inherits a new failure mode that needs messaging.

## Decision

**Option A**, with the `--staged` path getting staged-blob semantics (clone the real index into the temp index, commit that) and the auto-commit flows getting add-then-commit-from-temp-index semantics (which matches their current intent: they stage the files themselves immediately before committing). Option B's divergence check is retained ONLY as a cheap assertion inside the `--staged` path to produce a clear error message when the temp-index approach detects nothing staged.

Rationale: the engine is the foundation R6 and every quest/intent/workitem/dungeon/repair auto-commit builds on; in-repo docs already promise index semantics for `--staged`; a guard-only fix leaves the concurrency window open and converts daily workflows into errors.

## Consequences

- Task 07.01 implements the temp-index engine with table-driven and scratch-repo tests reproducing the N-4 cases (staged v2 / worktree v3 commits v2 under --staged; auto-commit captures the add-time snapshot).
- Task 07.02 (N-5) fixes `listStagedFiles` rename parsing against the same fixtures.
- Tasks 07.03/07.04 (R6/N-14) may then use `Only`-style scoping safely through the new engine.
- `pkg/commitkit` semantics notes updated; the fest coordination workitem (07.05) references this decision.
