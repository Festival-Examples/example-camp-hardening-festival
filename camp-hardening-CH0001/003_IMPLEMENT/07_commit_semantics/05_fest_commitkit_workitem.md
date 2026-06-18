---
fest_type: task
fest_id: 05_fest_commitkit_workitem.md
fest_name: fest_commitkit_workitem
fest_parent: 07_commit_semantics
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.714652-06:00
fest_updated: 2026-06-14T23:44:07.055384-06:00
fest_tracking: true
---


# Task: Create fest-project design workitem for commitkit mirror change (correction 2)

## Objective

Create a `design` workitem in the fest project via `camp workitem create` that documents the unscoped-commit pattern in `fest/internal/commands/commit/commit.go:541-553`, the required scoping mirror from CH0001's commitkit changes, and the commitkit semantic-change summary fest consumers need, so the fix is tracked and does not fall through the cracks after this festival closes.

## Requirements

- [ ] Run `camp workitem create --help` first to learn the current flags before constructing the create command.
- [ ] Create a `design` workitem in the fest project (not the camp project) describing the required change; the title clearly identifies the location (`fest/internal/commands/commit/commit.go:541-553`) and the action (scope the unscoped commit).
- [ ] The workitem body contains: the file:line reference, a description of what the code currently does wrong, what the required change is, the `commitkit` semantic-change summary, and a reference to D001 and festival CH0001.
- [ ] The workitem is listed by `camp workitem list` in the fest project after creation.
- [ ] No edits to any file under `projects/fest/` in this task or anywhere in this festival (C-5, C-6).

## Implementation

### Background: what the workitem must document

Second-pass correction 2 states:

> fest's exposure comes through its own `commit.go:541-553` reuse of commitkit primitives in the same unscoped pattern, not (only) through `SyncSubmoduleRef`. Fixing `SyncSubmoduleRef` alone does not close the fest side; the fest repo needs the same scoping change.

The fest code at `projects/fest/internal/commands/commit/commit.go:541-553` is `commitCampaignRoot`:

```go
func commitCampaignRoot(ctx context.Context, campaignRoot string, paths []string, commitMessage string) (string, error) {
    if err := commitkit.StageFiles(ctx, campaignRoot, paths...); err != nil { ... }

    hasChanges, err := commitkit.HasStagedChanges(ctx, campaignRoot)
    // ^^ whole-repo check -- sees ALL staged content, not just paths
    ...
    if err := commitkit.Commit(ctx, campaignRoot, commitkit.CommitOptions{
        Message: commitMessage,
    }); err != nil { ... }
    // ^^ unscoped commit -- sweeps everything staged, not just paths
    ...
}
```

The pattern is identical to the GIT-2 findings in camp: stage paths, check whole-repo staged, commit unscoped. The fix is to replace `HasStagedChanges` with a path-scoped check and replace `Commit` with `CommitScoped` (or the equivalent new commitkit entry point from task 03).

The workitem must capture enough detail that a fest developer can implement the fix without re-reading the full CH0001 review.

### Step 1: Learn the create command flags

Run this from the worktree (or campaign root; `camp workitem create` is cwd-agnostic via campaign detection):

```bash
camp workitem create --help
```

Note which flags are available: typically `--type`, `--title` or positional title, `--body` or `--body-file`, `--project`. Confirm whether `--project` accepts a project path like `projects/fest` or a name like `fest`.

### Step 2: Construct and run the create command

Based on the actual `--help` output, construct a command similar to:

```bash
camp workitem create \
  --type design \
  --project projects/fest \
  --title "scope unscoped commit in commitCampaignRoot (commitkit mirror for CH0001)" \
  --body "$(cat <<'EOF'
## Problem

fest/internal/commands/commit/commit.go:541-553 (function commitCampaignRoot)
reimplements the same unscoped-commit pattern that camp's GIT-2 finding covers:

  1. commitkit.StageFiles(ctx, campaignRoot, paths...)   -- stages only paths
  2. commitkit.HasStagedChanges(ctx, campaignRoot)       -- whole-repo check (wrong)
  3. commitkit.Commit(ctx, campaignRoot, ...)             -- unscoped commit (wrong)

This means: if the user or a concurrent process has any other content staged at
the campaign root, commitCampaignRoot will sweep it into the fest commit under
fest's tag and message. This is the same blast-radius as camp's GIT-2 finding.

## Required Change

1. Replace commitkit.HasStagedChanges (whole-repo) with a path-scoped check for
   the paths slice. After CH0001 lands, commitkit will export HasStagedPathChange
   (or equivalent) for this purpose.

2. Replace commitkit.Commit (unscoped) with commitkit.CommitScoped(ctx, campaignRoot,
   paths, commitkit.CommitOptions{Message: commitMessage}). CommitScoped is being
   added to pkg/commitkit by camp festival CH0001 task 03.

The fix must be applied consistently: the no-op guard (HasStagedChanges) and the
commit (Commit) must both be path-scoped.

## Commitkit Semantic Change Summary (for fest consumers)

camp festival CH0001 changes pkg/commitkit as follows:

- SyncSubmoduleRef: now commits scoped to projectRelPath only. Previously it
  committed the entire campaign root. The signature is unchanged. The no-op check
  is now path-scoped (not whole-repo).

- New export CommitScoped(ctx, repoPath string, paths []string, opts CommitOptions):
  commits only the given paths using a temp-index snapshot. Correct replacement for
  Commit when a specific file set must be committed without sweeping staged content.

- New export HasStagedPathChange(ctx, repoPath, path string) (bool, error):
  path-scoped alternative to HasStagedChanges. Use this for no-op guards that
  should only react to a specific path.

All existing function signatures are unchanged (backward compatible).

## References

- Decision record: festivals/planning/camp-hardening-CH0001/002_PLAN/decisions/D001_n4_only_commit_semantics.md
- Festival: camp-hardening-CH0001 (Sequence 07_commit_semantics, Task 05)
- Second-pass finding: second-pass-fable.md verification row 10, correction 2
- camp GIT-2 (R6): recommendations.md R6
EOF
)"
```

The exact flag syntax depends on what `--help` reports. If `--body` requires a file, write the body to a temp file and pass `--body-file`. If `--title` is positional, pass it as the first argument.

### Step 3: Verify the workitem was created

Run:

```bash
camp workitem list --project projects/fest
```

Confirm the new workitem appears with the correct title and type. If `--project` is not a valid filter flag for `workitem list`, check `camp workitem list --help` for the correct flag to scope the list to the fest project.

Also run:

```bash
camp workitem list --project projects/fest --type design
```

to confirm it is categorized as a design workitem.

### Step 4: Optionally record the workitem ID

Note the workitem ID in this task's completion notes so it is easy to reference in the fest project's backlog. The ID will be in the format the camp workitem system uses (typically `WI-<hex>`).

## Done When

- [ ] `camp workitem create --help` was run and the flags were read before constructing the command
- [ ] A design workitem exists for the fest project; `camp workitem list` (scoped to fest) shows it
- [ ] The workitem body contains: the file:line (`fest/internal/commands/commit/commit.go:541-553`), what the code currently does wrong, what the required change is (path-scoped HasStagedPathChange and CommitScoped), the commitkit semantic-change summary, and a reference to D001 and CH0001
- [ ] No file under `projects/fest/` was edited at any point in this task
- [ ] Both-profile gate still passes (this task makes no code changes; gate verification is a confirm-no-regression check)

## Out of Scope

- Any code change in the fest repository (C-5 hard constraint; fest-side fix is the fest team's workitem to execute).
- Implementing the fest fix in this festival. The workitem is the tracking artifact; execution happens in a future fest festival or sprint.
- Validating that the commitkit API changes in task 03 are backward-compatible at compile time against the fest codebase. That is a good-practice check noted in task 03; it is not this task's responsibility.