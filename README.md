# Camp Hardening Festival (CH0001) — Example

> Example Festival sourced from [Obedience-Corp/camp PR #324](https://github.com/Obedience-Corp/camp/pull/324).

## Summary

This PR carries the completed `camp-hardening` festival remediation branch for CH0001. The branch has been rebased onto current `origin/main`, is out of draft, and both rounds of review follow-up fixes have been pushed.

The festival is fully complete and promoted to `dungeon/completed`:

- Festival path: `festivals/dungeon/completed/2026-06-15/camp-hardening-CH0001`
- Festival status: `dungeon/completed`
- Progress: 100% - 4/4 phases, 12/12 sequences, 145/145 tasks

## Current Head

- Branch: `camp-hardening`
- Pushed head: `ec4d0542992dc9aad7970775043bc1992e9702aa`
- PR state: ready for review

## User-Visible Changes / Merge Expectations

This is a broad hardening PR, not a tiny internal-only patch. Users should expect `camp` to be more conservative and more explicit after merge.

### Safer destructive and disk-mutating commands

Several commands now refuse risky operations that previously could proceed:

- `camp cp` rejects same-file copies and directory self-recursion, including symlinked destination ancestors.
- `camp dungeon move` / docs routing rejects structural campaign directories such as `festivals` and `projects` even when named explicitly.
- `camp worktrees clean` has confirmation, noninteractive refusal, dirty-worktree checks, and safer stale worktree handling.
- `camp project remove`, `camp sync`, `camp pull`, `camp pull all`, `camp refs-sync`, and related project/git flows have stronger dirty-state, unpushed-ref, stale-ref, rollback, and rebase guards.
- `camp fresh` / project prune now preserves dirty branch worktrees instead of depending on git to reject branch deletion later.

Expected effect, narrowed: this PR does not broadly remove intentional noninteractive state mutation. Clean/default automation should still run without prompts. The explicit new or changed consent points are:

- `camp worktrees clean`: new confirmation gate before deletion. In non-TTY, deletion requires `--yes` or `--force`; `--yes` only skips confirmation. If `--force` targets a real worktree with uncommitted changes, it now refuses unless `--discard-dirty` is also passed. Full-clone entries whose `.git` is a directory are never auto-removed.
- `camp project prune`: existing `--force` still skips local branch deletion confirmation. New `--discard-dirty` is only for deleting dirty branch worktrees; without it those worktrees and branches are skipped and reported. `--remote-delete` still asks independently.
- `camp fresh`: remains deliberately noninteractive. It still bypasses prune confirmation, but it does not set `DiscardDirty`; dirty branch worktrees are preserved/skipped instead of deleted.
- `camp project remove`: existing `--force` now also overrides the new unpushed-ref guard for `.git/modules/projects/<name>`. Without `--force`, removal refuses when deleting the module gitdir would discard committed but unpushed branch history.
- `camp sync`: `--force` was already the preflight override. This PR makes dirty/unpushed/drift failures louder and safer; clean sync remains noninteractive.
- `camp refs-sync`: `--force` for staged-root changes already existed; this PR changes scoped committing and skip reporting, not its explicit-flag contract.
- `camp init --repair --yes`: this was already the scripting path; this PR only rewords and hardens repair/migration failures.

### Output and contract changes

Scripts and consumers may notice output/contract differences:

- JSON outputs are more consistently versioned and shaped, especially intents, flow items, concepts, doctor, lists, project lists, and workitem-related commands.
- `camp intent ... --json` and `camp intent ... -f json` now use the same JSON error envelope path, including flag-parse errors.
- `camp concepts --json` now includes nested `children`, matching the human recursive concept tree.
- Exit codes and command errors are more standardized. Scripts that match exact human error strings may need updates.
- CLI reference docs were regenerated and may show changed help text, aliases, and command grouping.

Expected effect: machine consumers should become more reliable long term, but exact-output tests or scripts may need small updates.

### Git, commit, pull, and status behavior

Git-backed workflows were hardened:

- `camp commit`, `camp p commit`, workitem commit, and refs sync use more careful scoping and staging behavior.
- `camp pull` / `pull all` and sync paths are more defensive around in-progress rebases, dirty worktrees, stale refs, and fallback checkout behavior.
- `camp status all` and related status collection were refactored behind shared internals.

Expected effect: normal clean happy paths should remain familiar; edge cases should fail earlier with clearer messages instead of mutating state ambiguously.

### Navigation, shell, shortcuts, and concepts

Navigation and shell helpers were refactored and tightened:

- Shell navigation wrappers are intended to be more transparent about argv and failures.
- Shortcut and nav-index resolution were consolidated.
- Concept listing now preserves configured nested children in both human and JSON surfaces.

Expected effect: day-to-day navigation should feel the same or more consistent; scripts using exact shortcut output should be checked.

### Intent, workflow, workitem, and quest handling

Workflow metadata paths have stronger safety behavior:

- Intent parsing, guarded writes, migration/read behavior, and JSON output were hardened.
- Workitem list/create/link/resolve/priority flows gained stricter references and more consistent output.
- Quest creation/list/slug behavior received validation and persistence fixes.
- Workflow state writes and migration/sync paths are more atomic and collision-aware.

Expected effect: fewer silent partial writes or accidental mutations. Reads that previously repaired or migrated opportunistically may now be more conservative.

### Developer workflow changes

If this repo has hooks installed, pushes now run a meaningful default local gate:

- whitespace check
- stable build
- dev build
- stable/dev/integration `go vet`
- stable lint/ratchets
- dev lint/ratchets
- dev-profile short unit tests

Heavier gates remain explicit:

- `CAMP_GATE_FAST=1 git push` runs `just gate-fast`
- `CAMP_GATE_FULL=1 git push` runs `just gate`

Expected effect: local pushes are slower than the thinnest smoke, but should catch real regressions while hosted CI is paused.

## Practical Risk Read

- Likelihood of some user-visible CLI change: high.
- Likelihood of a normal clean happy-path workflow breaking: moderate-low, but not zero.
- Likelihood that scripts depending on exact human output, error text, or old JSON shapes need updates: moderate to high.

This PR should be treated as a safety/contract hardening release. The intended direction is fewer ambiguous mutations, clearer failure modes, and more stable machine-readable surfaces.

## Review Follow-up

The latest pushes address the requested-change threads by:

- rejecting structural triage exclusions such as `festivals` across explicit dungeon/docs move paths
- converting the reviewed new raw `fmt.Errorf` returns to `camperrors`
- replacing the file-level `fmt.Errorf` allowlist with a merge-base count ratchet
- resolving `camp cp` symlinked destination ancestors before self-recursion checks
- preserving dirty branch worktrees from final branch deletion during prune
- emitting JSON error envelopes for `intent` commands when `-f json` is used with flag errors
- preserving nested concept children in `concepts/v1alpha1`
- strengthening `gate-push` with vet, dev lint, and dev short tests
- making the EXDEV copy fallback handle read-only directories safely
- removing the inert skipped prune fake-executor tests

## Current Validation

Validation on the current head:

- `GOFLAGS=-buildvcs=false go test ./cmd/camp ./cmd/camp/intent ./internal/statusmove ./internal/leverage ./internal/state ./internal/prune -count=1`: passed
- `GOFLAGS=-buildvcs=false go vet ./cmd/camp ./cmd/camp/intent ./internal/statusmove ./internal/leverage ./internal/state ./internal/prune`: passed
- `git diff --check`: passed
- `just lint-no-fmt-errorf && just test-ratchet`: passed
- `GOFLAGS=-buildvcs=false just gate-push`: passed

Festival closeout validation before the final rebase also passed the broader local gates, integration suite, docs generation, and `fest validate`.

## Closeout Notes

- PR #323 was the earlier agent-authored draft and is closed. This active PR is #324, authored by `lancekrogers`.
- Branch pushes used `--no-verify` only after equivalent local validation, including the final `gate-push` pass.
- Campaign-root festival metadata was intentionally not pushed from this branch because campaign root `main` has unrelated local state outside this camp PR.
- Completion artifacts were written under the completed festival: `VERIFICATION_CHECKLIST.md`, `DISPOSITION_LOG.md`, and `COMPLETION_REPORT.md`.


