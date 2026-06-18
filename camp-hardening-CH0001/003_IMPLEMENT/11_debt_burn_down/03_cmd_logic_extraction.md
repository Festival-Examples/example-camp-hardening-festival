---
fest_type: task
fest_id: 03_cmd_logic_extraction.md
fest_name: cmd_logic_extraction
fest_parent: 11_debt_burn_down
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.78757-06:00
fest_updated: 2026-06-15T04:40:56.94741-06:00
fest_tracking: true
---


# Task: Command-Tree Convention Documentation and Cmd Logic Extraction

## Objective

Document Tree B constructor style as the canonical command pattern, then extract the largest business logic clusters from `cmd/` into `internal/` packages in extraction-only commits that make zero behavior changes.

## Requirements

- [ ] Task 02 (dead code sweep) has completed before this task starts; the dependency is load-bearing (R29 strictly before R30)
- [ ] Tree B constructor style documented in a repo-level conventions file
- [ ] Each extraction is a separate commit; no behavior changes inside any extraction commit
- [ ] Existing tests remain green after every extraction commit (both profiles)
- [ ] ARCH-7: extracted domain code returns structs; rendering stays in `cmd/`
- [ ] ARCH-3 addressed by package-level doc comments cross-referencing the `flow`/`workflow` naming inversion
- [ ] `EnsureRefForCommit` moved from `internal/commands/workitem` into `internal/workitem`
- [ ] `cmd/camp/run.go` duplicated resolution consolidated onto extracted nav resolution

## Implementation

### Background

ARCH-1 and ARCH-2 document two parallel cobra command trees and extensive business logic stranded in `cmd/`. Tree A uses package-level `var` plus `init()` registration; Tree B uses constructor functions (e.g., `NewWorkitemCommand()`). Tree B is the newer, testable style. This task documents that decision and begins migrating the worst ARCH-6 file-size offenders.

All extraction commits must be pure refactors: zero behavior change, verified by the unchanged test suite. No new tests are required inside the extraction commits themselves, but existing tests must pass. New tests covering the extracted behavior belong in the receiving `internal/` package and can be added in a follow-on commit within this task.

### Step 1: Document Tree B as canonical

Check where existing conventions documentation lives:

```bash
ls CONTRIBUTING.md docs/contributing* docs/architecture* docs/conventions* 2>/dev/null
```

If `CONTRIBUTING.md` exists, add a section. If not, create `docs/architecture/command-conventions.md`. The content:

```markdown
## Command Registration: Use Tree B Constructor Style

New cobra commands must use the constructor function pattern (Tree B):

- Command defined as a function returning `*cobra.Command` (e.g., `NewFooCommand() *cobra.Command`)
- Wired at `cmd/camp/root.go` or via a `Register(root *cobra.Command)` function
- No package-level `var cmdFoo *cobra.Command`; no `init()` registration
- Error handling via `camperrors`; JSON output via `jsoncontract`
- Exit codes via `camperrors.CommandError`

Examples of the pattern: `internal/commands/workitem`, `internal/commands/workflow`.

Tree A (package-level vars + `init()`) exists in older code under `cmd/camp/` subpackages. It is being migrated incrementally; do not add new Tree A commands.

## flow vs workflow naming

`internal/workflow` is the status-directory state machine (`.workflow.yaml`, transitions).
`internal/flow` is a shell-command registry/runner (`.campaign/flows/registry.yaml`).
The CLI command `camp flow` is mostly backed by `internal/workflow`; `camp workflow` manages workflow collections.
These names are inverted relative to what a reader expects. Until they are renamed, both packages carry a doc comment cross-referencing the other.
```

Add cross-reference doc comments to both packages:

`internal/workflow/service.go` (or the package doc file): add a comment at the top stating this package is backing `camp flow` commands (not `camp workflow`) and cross-reference `internal/flow`.

`internal/flow/registry.go`: add a comment stating this package is the shell-command runner used by `camp flow run/list`, distinct from `internal/workflow`.

### Step 2: `cmd/camp/pull.go` extraction

Target: move `runGitPullWithLockRetry`, `classifyPullCommandError`, `runPullAll`, `pullSingleTarget` into `internal/pull` (or extend `internal/git` if that package is the better home; use `internal/pull` since the logic is specific to the pull-all orchestration).

Verified targets in the worktree:
- Lock retry wrapper: `runGitPullWithLockRetry` (around line 91)
- Error classification: `classifyPullCommandError` (around line 119)
- Pull-all engine: `runPullAll`, `pullSingleTarget` (around lines 170-305)

The rebase abort logic at `pull.go:305-314` is a correctness fix covered by sequence 03; if sequence 03 already landed, the corrected version is what gets extracted. If not, extract the current code and note that the behavior fix follows in the same extraction.

Create `internal/pull/pull.go`:
```go
package pull

import (
    // import only what the extracted functions need
)

// PullOptions configures a pull-all operation.
type PullOptions struct {
    NoRecurse     bool
    DefaultBranch bool
    // ... carry over from cmd/camp/pull.go PullAllOptions
}

// PullResult carries per-repo outcomes.
type PullResult struct {
    // carry over existing result type fields
}

// RunPullAll pulls all submodules in a campaign.
func RunPullAll(ctx context.Context, campaignRoot string, opts PullOptions) error { ... }
```

Keep `cmd/camp/pull.go`'s cobra command wiring thin: it parses flags, builds `pull.PullOptions`, calls `pull.RunPullAll(ctx, root, opts)`, and handles the output.

One commit: `refactor(pull): extract pull-all engine from cmd/ into internal/pull`.

### Step 3: `cmd/camp/status_all.go` extraction

Verified targets:
- `getRepoStatus` at `:176`
- `renderStatusTable` at `:285`
- `writeStatusCache` at `:391`

Create `internal/status/status.go` with the domain types and `GetRepoStatus`, `WriteStatusCache`. Keep `renderStatusTable` in `cmd/camp/status_all.go` per ARCH-7 (rendering stays in cmd). The function `getRepoStatus` returns a `status.RepoStatus` struct; `renderStatusTable` takes `[]status.RepoStatus`.

Note: `getRepoStatus` calls `internal/git/query.go:Output` which uses `strings.TrimSpace`. Task 04 fixes the porcelain parsing. This extraction does not change that behavior.

One commit: `refactor(status): extract repo status collection from cmd/ into internal/status`.

### Step 4: `cmd/camp/navigation/shortcuts.go` extraction

Verified targets:
- `computeShortcutDiff` at `:617`
- `runShortcutsReset` at `:731`
- `runShortcutsAddJump` at `:477`

Note: `internal/shortcuts` already exists. Post dead-code sweep (task 02), `Expander` and `FeedbackWriter` are gone. The package is now sparse. Add the extracted diff/reset engine there.

Create or extend `internal/shortcuts/diff.go`:
- `ComputeShortcutDiff(current, defaults map[string]config.ShortcutConfig) ShortcutDiff`

Create or extend `internal/shortcuts/reset.go`:
- `RunReset(ctx context.Context, opts ResetOptions) error`

`runShortcutsAddJump` at `:477-581` is the add-jump orchestration; move its domain logic (validation, conflict checking) to `internal/shortcuts/add.go` leaving only flag parsing and output in `cmd/`.

Per ARCH-7: the extracted functions return structs; the cobra commands in `cmd/camp/navigation/shortcuts.go` handle rendering.

Target: `cmd/camp/navigation/shortcuts.go` shrinks from 890 toward the 500-line standard.

One commit per sub-extraction (diff, reset, add-jump) if they are large enough to warrant it; one commit if they are small.

### Step 5: `cmd/camp/navigation/go.go` resolution helpers

Verified targets:
- `handleRelativePathNavigation` at `:339`
- `resolveRelativePathNavigation` at `:367`
- `fuzzyResolveDirectory` at `:405`

Move into `internal/nav/resolve.go` (create if absent). These are pure resolution semantics with no output; the extracted functions return `(string, error)`.

One commit: `refactor(nav): extract resolution helpers from go.go into internal/nav`.

### Step 6: `cmd/camp/leverage/helpers.go` extraction

Verified targets:
- `computeProjectScore` at `:256`
- `persistCurrentSnapshots` at `:322`

Move into `internal/leverage`. These are the scoring policy and snapshot persistence. The `internal/leverage` package already exists (authors, leverage types). Add `internal/leverage/score.go` and `internal/leverage/snapshot.go`.

One commit: `refactor(leverage): extract scoring and snapshot logic from cmd/ into internal/leverage`.

### Step 7: `cmd/camp/dungeon/crawl.go` staging policy extraction

Verified targets:
- `commitCrawlChanges` at `:165`
- `stageTrackedCrawlSourceDeletions` at `:325`
- `isSafeCrawlCommitPath` at `:419`

Move into `internal/dungeon/crawl.go` (create if absent). The `internal/dungeon` package already exists.

One commit: `refactor(dungeon): extract crawl staging policy from cmd/ into internal/dungeon`.

### Step 8: `EnsureRefForCommit` into `internal/workitem`

Currently at `internal/commands/workitem/staging_branches.go:205-207`. Referenced from `cmd/camp/commit_workitem.go:30` (Tree A command importing a Tree B package, i.e., cross-tree coupling).

Move `EnsureRefForCommit` and its helpers into `internal/workitem/ref.go` or extend the existing `internal/workitem/ref.go`. Update the import in `cmd/camp/commit_workitem.go`.

One commit: `refactor(workitem): move EnsureRefForCommit from internal/commands into internal/workitem`.

### Step 9: `cmd/camp/run.go` duplicated resolution consolidation

`cmd/camp/run.go:96-165` re-implements shortcut and sub-shortcut resolution including a redundant double `index.Resolve` probe. After step 5's extraction of `internal/nav` resolution helpers, replace `run.go:96-165` with calls to the extracted helpers.

One commit: `refactor(run): consolidate duplicated nav resolution onto internal/nav helpers`.

### Step 10: `cmd/camp/init/command.go` RunFlow split

`RunFlow` at `:151-353` is ~200 lines. Extract the skill projection work (`projectCampaignSkills` at `:383`) into `internal/skills/projection.go` (the file already exists per task 08). Extract the flow-runner call coordination into `internal/flow`. The cobra command in `init/command.go` becomes a thin orchestrator.

This extraction is lower priority than steps 2-9. If it does not fit in the same sprint, note that `init/command.go` remains at 415 lines and flag for sequence 12.

### Acceptance

For ARCH-6 file-size targets touched by this task, verify the line count is moving toward (not necessarily below) the 500-line standard after extraction:

```bash
wc -l cmd/camp/pull.go cmd/camp/status_all.go \
       cmd/camp/navigation/shortcuts.go cmd/camp/navigation/go.go \
       cmd/camp/leverage/helpers.go cmd/camp/dungeon/crawl.go \
       cmd/camp/init/command.go
```

### Out of scope

- Extracting logic from files not listed above
- Changing behavior inside any extraction commit (pure mechanical refactor only)
- Renaming packages to fix the ARCH-3 flow/workflow inversion (noted as out-of-scope in the plan; package doc cross-references are the mitigation)
- Tree A to Tree B migration for individual commands (that is a sequence 12 polish concern)

## Done When

- [ ] `docs/architecture/command-conventions.md` (or CONTRIBUTING.md section) documents Tree B as canonical and includes the flow/workflow naming note
- [ ] Package doc comments in `internal/workflow` and `internal/flow` cross-reference each other
- [ ] `EnsureRefForCommit` lives in `internal/workitem`, not `internal/commands/workitem`
- [ ] `cmd/camp/commit_workitem.go` imports from `internal/workitem`
- [ ] `cmd/camp/pull.go`, `cmd/camp/status_all.go`, `cmd/camp/navigation/shortcuts.go` are measurably smaller after extraction
- [ ] Each extraction is a separate commit with no behavior change
- [ ] `just build`, `BUILD_TAGS=dev just build`, `go test ./...`, and `go test -tags dev ./...` pass after every extraction commit