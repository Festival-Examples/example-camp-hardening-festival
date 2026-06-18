# Gap Analysis: camp-hardening (CH0001) Planning

Gaps and open decisions identified after reviewing 001_INGEST/output_specs/ against the planning phase goal. Each item names where it gets resolved (decision record, verification step, or sequence design).

## Open decisions (need decision records in decisions/)

### GAP-1. N-4 fix approach: index snapshot vs divergence guard
The `--only` commit engine commits worktree content, not the index. Two candidate fixes:
- (a) Temporary `GIT_INDEX_FILE` built via `read-tree HEAD` plus `add -- <paths>`: true snapshot semantics, larger change, touches every auto-commit call site's assumptions.
- (b) `git diff --quiet -- <paths>` index==worktree verification before `--only`, failing loudly on divergence: smaller change, but turns a silent over-commit into a hard error for `workitem commit --staged` users with unstaged edits to staged files.
Resolution: decision record DEC-001. The choice gates the design of REQ-19's sequence and affects N-5's test fixtures.

### GAP-2. R9 flow channel: promote `flow` to stable vs channel-aware templates
Scaffolded skills instruct `camp flow ...` but `flow` is dev-only. Options:
- (a) Promote `flow` to stable: simplest user story; but `internal/commands/flow` has zero tests (TEST-4) and N-7's dead `--json` shows the surface is immature.
- (b) Channel-aware scaffolding: templates render flow instructions only for dev builds; README and `dungeon add` help scrubbed; stable users never see the commands.
Resolution: decision record DEC-002. Affects the launch-docs sequence and possibly the contract sequence (N-7).

### GAP-3. R25 shared statusdir move primitive: before or after the P0 migration fixes
The P0 migration fixes (REQ-06) and the primitive (REQ-32) overlap: building the primitive first would let REQ-06 consume it, but P0 cannot wait on a P3-sized unification. Resolution: decision record DEC-003 (expected: targeted P0 fixes first with the primitive extracted later in cleanup, REQ-06 code written so the later migration is mechanical).

### GAP-4. Schema-bump list and festival-app notification path
Which JSON changes bump schema_version: intent envelope (new contract), workitem stage vocabulary (R26 bump), workitem omitempty fix (JSON-3, rides R26's bump or next), status-all versioning, version --json profile field, manifest annotation additions (int version field semantics). Notification: docs/json-contracts.md rows plus a festival-app coordination workitem. Resolution: decision record DEC-004.

### GAP-5. Pre-push hook content and installation mechanism
What the hook runs (candidate: both-profile build plus unit tests plus lint, or a faster subset with the full matrix in `just gate`), how it is installed (committed hook file plus `just hooks install` vs core.hooksPath), and how it stays fast enough to not get bypassed. Resolution: decision record DEC-005, consumed by the gates sequence design.

### GAP-6. POLISH phase scope vs IMPLEMENT tail
The R29-R36 sweeps could live at the tail of IMPLEMENT or in POLISH. fest.yaml already defines IMPLEMENT and POLISH phases. Resolution: decision record DEC-006 (expected: all code-changing sweeps in IMPLEMENT so POLISH stays a pure verification/closeout phase per FESTIVAL_GOAL's phase list).

## Verification gaps (resolve during DECOMPOSE/DESIGN, not decisions)

### GAP-7. File:line drift between review baselines and the worktree
Review citations are from camp 5c22d2b (first pass) and 11c270f (second pass); the linked worktree branches from bc2ad1f. Every task document must verify its cited targets against `projects/worktrees/camp/camp-hardening` (read-only) during planning and note drift. Two commits between baselines were intent TUI work; drift risk concentrates in `internal/intent/tui/` adjacent files.

### GAP-8. Dirty-flag naming for destructive commands
REQ-01/REQ-03 require "an explicit flag to destroy dirty trees". The flag name and semantics (`--force` overload vs a new `--discard-dirty` style flag) must be consistent across worktrees clean, fresh, and prune. Resolve in the data-loss sequence design; record in the sequence docs.

### GAP-9. R22 exit-code table specifics
The CLI-wide table (0 ok, 1 runtime, 2 usage, 3+ documented) conflicts with three existing per-command tables (doctor, sync, clone) that agents may already parse. camp is unreleased publicly (no external users), so breaking these is acceptable; the table contents and doc location need fixing in the cleanup sequence design.

## Non-gaps (resolved by constraints, no action)

- Hosted CI: ruled out by C-1; not a decision.
- fest-side commitkit mirror: ruled out of scope by C-5; the workitem content is planned in the contracts/commit sequence.
- Verified-negatives: C-3 settles them; no re-investigation tasks.
- Host t.TempDir vs container conventions: C-9 settles test placement.
