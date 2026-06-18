# Constraints: camp-hardening (CH0001)

## Hard constraints (non-negotiable)

### C-1. Local gates only; GitHub Actions is intentionally paused
- No `.github/workflows/` files are added or modified anywhere in the diff. All gate enforcement is `just` recipes plus a local pre-push git hook.
- Why: hosted CI is paused as a deliberate policy decision. The review's original R13 wording ("add minimal CI") is superseded by the second-pass section 4.7 framing: `just test` both profiles, `go vet -tags=integration ./...` in `just lint`, and a pre-push hook. Festival planning and execution must never reference re-enabling hosted CI.

### C-2. Both build profiles at every gate
- Every implementation gate runs `just build`, `just test`, and `just lint` in BOTH profiles: stable (no tags) and `BUILD_TAGS=dev`.
- Why: the manifest regression (TEST-1) shipped precisely because dev-tagged tests never ran; festival-app bundles DEV-profile camp, so the untested surface is the one the app actually executes. Running one profile and assuming the other is how this class of bug ships.

### C-3. Verified-negatives are do-not-fix
- The second pass's "Hunted and found clean" list (second-pass-fable.md section 3 tail) and the first pass's "Verified accurate (no action)" lists are settled. Do not re-litigate or "fix" them: intent filesystem migration guards, quest Rename/Restore rollback, attach marker handling, workitem links WithLock and QuarantineBroken, leverage reset/backfill boundaries, gendocs glob scope, project add/new creation guards, sync orphan-gitlink cleanup, shortcuts reset confirmation flow, commitkit message injection, NormalizeFiles path escaping, ExpandTrackedPaths rename parsing (correct in isolation; N-5 is the caller's scoping), worktree add/remove plumbing, push-all probe classification, workitem/workflow JSON determinism and nil-guards, jsoncontract `--json=false` handling, quest empty-list `[]`, no ANSI bleed, no map-order nondeterminism; push.go wording; the sync/stage/commit/mv/fresh/dungeon-add/promote Long texts.
- Why: re-fixing verified-clean code wastes gate cycles and risks regressions in code the review certified.

### C-4. Context facts honored, not re-litigated
- dev/rc/stable channel gating is intentional design (quest dev-only on purpose); findings are about execution inconsistencies, not the design.
- Registry path migration is complete; no compat shims for pre-rename paths.
- camp owns campaigns, fest owns festivals; boundary violations are camp bugs to fix, but fest's own behavior is out of scope.
- scc `@latest` in the integration harness is a deliberate documented policy choice.
- The host-git unit-test convention (34-file allowlist with a ratchet) is the current accepted state; the requirement is to start the burn-down, not to containerize everything at once.

### C-5. commitkit is fest's public API; fest-side changes are out of scope
- The R6/N-4 scoping change to `pkg/commitkit.SyncSubmoduleRef` is a fest-facing contract change and must be treated as such (compat note, careful semantics).
- fest's own `internal/commands/commit/commit.go:541-553` reimplements the same unscoped pattern; per second-pass correction 2 the fest exposure is independent of `SyncSubmoduleRef`. The festival captures this as a fest workitem (`camp workitem create` for the fest project) and does NOT edit the fest repo.

### C-6. Commit discipline
- Every commit during festival execution uses `fest commit` so the FE-CH0001 tag lands in history. Never raw `git commit`, never `camp commit`/`camp p commit` mid-festival.
- Implementation work happens on the linked worktree branch (`projects/worktrees/camp/camp-hardening`); planning never modifies the camp repo or worktree.

### C-7. No new dependencies without justification
- Per campaign standards, prefer stdlib and existing in-repo primitives (`fsutil`, `camperrors`, `jsoncontract`, the container harness). Any new external dependency (for example a lint tool beyond what `just lint` already drives) needs explicit justification in the task doc and Lance's approval.

## Process constraints

### C-8. Dependency ordering inside the festival
- Data-loss fixes land first; everything else builds on a tree that cannot destroy data.
- Gate work (R1, R13-R15) lands before the broad sweeps so every later sequence is verified by trustworthy gates in both profiles.
- N-4 and N-5 land BEFORE or WITH R6 (second-pass correction 3): the `Only` pattern's worktree semantics must be fixed before more call sites adopt it.
- R29 (dead code) lands STRICTLY BEFORE R30 (cmd-logic extraction) so nothing dead gets migrated.
- R20/R21/R26 and the N-7/N-9/N-15/N-16 contract fixes coordinate with festival-app: schema bumps are versioned, never silent shape changes.

### C-9. Test placement conventions
- New unit tests follow the repo's host `t.TempDir()` convention (the documented exception to the campaign container rule for camp/fest unit suites); destructive-path and filesystem-mutation coverage goes into `tests/integration/` (testcontainers, `//go:build integration`).
- The `lint-no-host-fs-tests` ratchet must not grow: no NEW host-git test files outside the 34-file allowlist.

### C-10. Schema and contract changes
- JSON shape changes ship as schema_version bumps with changelog entries in `docs/json-contracts.md`; consumers (festival-app) key on the version string.
- The workitem v1alpha5 omitempty fix (JSON-3) rides the next scheduled schema bump, not a silent change.

### C-11. Behavior changes to destructive commands need flags, not silent semantics swaps
- Where a fix changes what a command will delete (worktrees clean, fresh, prune, project remove, sync), the new safe behavior is the default and destruction requires an explicit flag; help text updated in the same change.

## Technical constraints

### C-12. Repo facts that bound the work
- camp is 917 Go files, ~186k lines; Go with cobra; module main package at `./cmd/camp`.
- Existing primitives to reuse, not duplicate: `fsutil.WriteFileAtomically` and `AcquireFileLock`, `internal/git` scoped-commit engine, `internal/jsoncontract`, `camperrors`, `statuspath.ExistingItemPath`, the `-z` porcelain parser at `internal/commands/workitem/staging_git.go:98-117`, `links.WithLock` as the locking pattern.
- The build system is modular justfiles driving `internal/buildutil`; `tasks.Build` honors `BUILD_TAGS`, `tasks.Test` does not yet (that gap is REQ-15's target).
- macOS `/var` -> `/private/var` symlinking affects every path-comparison fix; use `EvalSymlinks` patterns already present in `internal/pathutil`.

### C-13. File:line drift
- The review cites file:line at camp commits `5c22d2b` (first pass) and `11c270f` (second pass). The linked worktree branch may have drifted; every task doc must verify its targets against the worktree before editing, and implementers must re-locate by symbol if lines moved.
