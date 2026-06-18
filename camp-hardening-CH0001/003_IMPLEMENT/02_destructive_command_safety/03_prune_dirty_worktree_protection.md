---
fest_type: task
fest_id: 03_prune_dirty_worktree_protection.md
fest_name: prune_dirty_worktree_protection
fest_parent: 02_destructive_command_safety
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.619883-06:00
fest_updated: 2026-06-12T13:54:48.711033-06:00
fest_tracking: true
---


# Task: Prune Dirty Worktree Protection

## Objective

Apply the dirty-worktree check that already exists for detached-worktree removal to the branch-worktree removal path in `deleteLocalBranches`, introduce an explicit `--discard-dirty` flag required to destroy dirty worktrees, and stop `camp fresh` from passing unconditional `Force: true` past the dirty check (N-2).

## Requirements

- [ ] `deleteLocalBranches` in `internal/prune/prune.go` runs `detachedWorktreeClean` on each branch entry that has an active worktree before calling `wt.Remove`; dirty worktrees produce a `Result` with `Status: StatusSkipped` and `SkipReason: SkipReasonDirtyWorktree` instead of being force-removed.
- [ ] A new `DiscardDirty bool` field is added to `prune.Options`; only when `DiscardDirty: true` is the dirty check bypassed for branch worktrees.
- [ ] The interactive confirmation prompt in `deleteLocalBranches` (the existing `[y/N]` block) mentions dirty-worktree entries so the user knows some branches were skipped, and the prompt only appears when there are actually branches to delete (not just dirty-skipped ones).
- [ ] `camp fresh`'s prune call in `internal/commands/fresh/fresh.go:217-227` no longer sets `Force: true` unconditionally; it sets `Force: true` (skip the interactive confirmation because fresh is deliberate) but does NOT set `DiscardDirty: true`, so dirty worktrees are preserved during `camp fresh`.
- [ ] Help text for `camp project prune` and `camp fresh` documents the new `--discard-dirty` flag behavior and explains that dirty worktrees are skipped by default.
- [ ] Unit tests in `internal/prune/prune_dirty_test.go` (host `t.TempDir()` convention) demonstrate the dirty-skip and discard-dirty behavior using a fake executor (see task 04 for the full executor seam; this task adds the dirty-specific tests once the seam exists, or adds a thin in-package test helper if the seam does not exist yet).

## Implementation

### Background: the oversight

`deleteLocalBranches` in `internal/prune/prune.go` handles merged branches. When a branch has an active worktree, the code at lines 471-495 calls `wt.Remove(ctx, entry.Path, true)` with `force` hardcoded to `true`. The `detachedWorktreeClean` function at line 361-367 exists and is called for detached worktrees (lines 302-323), where it checks `git.HasChanges` and produces `SkipReasonDirtyWorktree`. The identical check is simply absent for the branch path. This is confirmed as an oversight by the comment structure: the detached path comment at line 301 reads `// dirty check` and the branch removal at line 471 has no such comment.

The `fresh.go` call at lines 217-227 (verified in worktree):

```go
pruneOpts := prune.Options{
    DryRun: opts.dryRun,
    Force:  true, // Skip confirmation — fresh is deliberate
    ...
}
```

`Force: true` has two effects in the current code: (1) skips the `[y/N]` confirmation in `deleteLocalBranches` (intended for fresh), and (2) in the original design passes `force=true` to `wt.Remove`. After this task, `Force` retains meaning (1) only; the dirty check runs regardless of `Force` unless the new `DiscardDirty` is also true.

### Step 1: Add `DiscardDirty` to `prune.Options`

In `internal/prune/prune.go`, add to the `Options` struct (after the existing fields):

```go
// DiscardDirty allows removal of branch worktrees that have uncommitted
// changes. When false (the default), dirty branch worktrees are skipped
// with SkipReasonDirtyWorktree. Detached worktrees always have their dirty
// check applied regardless of this flag.
DiscardDirty bool
```

### Step 2: Apply dirty check in `deleteLocalBranches`

Find the `worktreesToRemove` iteration in `deleteLocalBranches` (around lines 471-495). Currently:

```go
for branch, entry := range worktreesToRemove {
    if opts.DryRun {
        pr.Results = append(pr.Results, Result{
            Branch: branch,
            Status: StatusWorktreeWouldRemove,
            Detail: entry.Path,
        })
        continue
    }
    if err := wt.Remove(ctx, entry.Path, true); err != nil {
        ...
    }
}
```

Replace with:

```go
for branch, entry := range worktreesToRemove {
    if opts.DryRun {
        // For dry-run, still check dirtiness so the preview is accurate.
        if !opts.DiscardDirty {
            clean, err := detachedWorktreeClean(ctx, entry.Path)
            if err == nil && !clean {
                pr.Results = append(pr.Results, Result{
                    Branch:     branch,
                    Status:     StatusSkipped,
                    Detail:     fmt.Sprintf("would keep dirty worktree: %s", entry.Path),
                    SkipReason: SkipReasonDirtyWorktree,
                })
                continue
            }
        }
        pr.Results = append(pr.Results, Result{
            Branch: branch,
            Status: StatusWorktreeWouldRemove,
            Detail: entry.Path,
        })
        continue
    }

    // Non-dry-run: check dirtiness before calling git worktree remove --force.
    if !opts.DiscardDirty {
        clean, err := detachedWorktreeClean(ctx, entry.Path)
        if err != nil {
            pr.Results = append(pr.Results, Result{
                Branch: branch,
                Status: StatusError,
                Detail: fmt.Sprintf("dirty check for %s: %s", entry.Path, err),
            })
            continue
        }
        if !clean {
            detail := fmt.Sprintf("dirty worktree preserved: %s (use --discard-dirty to override)", entry.Path)
            pr.Results = append(pr.Results, Result{
                Branch:     branch,
                Status:     StatusSkipped,
                Detail:     detail,
                SkipReason: SkipReasonDirtyWorktree,
            })
            continue
        }
    }

    if err := wt.Remove(ctx, entry.Path, true); err != nil {
        pr.Results = append(pr.Results, Result{
            Branch: branch,
            Status: StatusError,
            Detail: fmt.Sprintf("worktree remove: %s", err),
        })
    } else {
        pr.Results = append(pr.Results, Result{
            Branch: branch,
            Status: StatusWorktreeRemoved,
            Detail: entry.Path,
        })
    }
}
```

### Step 3: Update confirmation prompt to mention dirty skips

In the `deleteLocalBranches` pre-deletion prompt section (around lines 442-469), after building `branchesToDelete` and `worktreesToRemove`, compute the dirty-skipped count and include it in the prompt:

The prompt block runs when `!opts.DryRun && !opts.Force`. The dirty check at this point has already happened (during the loop building `worktreesToRemove`). Add a count of dirty-skipped entries to the prompt output:

```go
fmt.Printf("\n%s Will delete %d merged or gone-upstream branch(es) in %s:\n",
    ui.WarningIcon(), len(branchesToDelete), ui.Value(pr.Name))
for _, b := range branchesToDelete {
    ...
}
// Note: dirty-worktree branches are already excluded from branchesToDelete
// at this point (they were added to pr.Results as skipped above and not
// added to branchesToDelete). The prompt reflects only what will actually
// be deleted.
```

Note: the dirty-skip decision must happen BEFORE building `branchesToDelete` so that dirty branches are excluded from the deletion list. Restructure the loop in `deleteLocalBranches` so dirtiness is checked during the initial scan, not after. Specifically: move the `worktreesToRemove` dirty check into the initial branch-classification loop (the `for _, branch := range merged` block at around line 416) so that dirty-worktree branches are classified as skipped before they enter `branchesToDelete`.

Revised classification loop:

```go
for _, branch := range merged {
    entry, hasWT := wtMap[branch]
    if hasWT && opts.SkipWorktreeBranches {
        ...skipped with SkipReasonActiveWorktree...
        continue
    }

    // Dirty check for branches with active worktrees.
    if hasWT && !opts.DiscardDirty {
        clean, err := detachedWorktreeClean(ctx, entry.Path)
        if err != nil {
            pr.Results = append(pr.Results, Result{
                Branch: branch,
                Status: StatusError,
                Detail: fmt.Sprintf("dirty check for %s: %s", entry.Path, err),
            })
            continue
        }
        if !clean {
            detail := fmt.Sprintf("dirty worktree: %s (use --discard-dirty to override)", entry.Path)
            if opts.DryRun {
                detail = "would keep: " + detail
            }
            pr.Results = append(pr.Results, Result{
                Branch:     branch,
                Status:     StatusSkipped,
                Detail:     detail,
                SkipReason: SkipReasonDirtyWorktree,
            })
            continue
        }
    }

    branchesToDelete = append(branchesToDelete, branch)
    if hasWT {
        worktreesToRemove[branch] = entry
    }
}
```

With this restructuring, `worktreesToRemove` only contains clean (or DiscardDirty) entries, and the later `wt.Remove` loop does not need to re-check dirtiness.

### Step 4: Update `camp fresh` to not set `DiscardDirty`

In `internal/commands/fresh/fresh.go`, the prune call (lines 217-227 in the worktree):

```go
pruneOpts := prune.Options{
    DryRun: opts.dryRun,
    Force:  true, // Skip confirmation — fresh is deliberate
    Remote: opts.pruneRemote,
    BaseRef:       syncState.baseRef,
    RefreshRemote: true,
}
```

This remains correct as-is after step 1. `Force: true` continues to skip the interactive `[y/N]` prompt (appropriate: fresh is deliberate). `DiscardDirty` defaults to `false`, so dirty worktrees are preserved. No change needed to this block, but verify it after step 1 compiles.

If the `fresh all` subcommand has a separate prune call, apply the same review.

### Step 5: Wire `--discard-dirty` in the `camp project prune` command

Locate the `camp project prune` command (likely in `cmd/camp/project/` or a subcommand file). Add the flag:

```go
pruneCmd.Flags().BoolVar(&pruneDiscardDirty, "discard-dirty", false,
    "Allow removal of worktrees with uncommitted changes (default: dirty worktrees are preserved)")
```

Pass it through to `prune.Options{DiscardDirty: pruneDiscardDirty}`.

### Step 6: Update help text

In the `Long` field of the prune command, add a sentence:

> Branch worktrees with uncommitted changes are skipped by default. Pass --discard-dirty to remove them.

In the `fresh` command's `Long` field, add:

> Branch worktrees with uncommitted changes are preserved during prune. Run `camp project prune --discard-dirty` to override.

### Unit test design notes for task 04

Task 04 adds the full prune test suite. This task's dirty-protection logic should be specifically covered there. The design notes for the test seam (to inform task 04):

The key classification logic is now entirely in `deleteLocalBranches` (local) and `Execute` (orchestrator). For unit-testability, introduce a minimal `Executor` interface:

```go
type Executor interface {
    MergedBranches(ctx context.Context, path, baseRef string) ([]string, error)
    GoneBranches(ctx context.Context, path string) ([]string, error)
    HasChanges(ctx context.Context, path string) (bool, error)
    DeleteBranch(ctx context.Context, path, branch string) error
    DeleteBranchForce(ctx context.Context, path, branch string) error
    RemoveWorktree(ctx context.Context, projectPath, worktreePath string, force bool) error
}
```

Or alternatively, wrap the classification logic (candidate selection, dirty check, result building) in functions that take a `HasChanges func(ctx, path) (bool, error)` parameter, leaving the git calls injectable. This avoids a broad interface but covers the dirty-check seam needed for task 04's tests.

## Done When

- [ ] All requirements met
- [ ] `just build` and `BUILD_TAGS=dev just build` pass from the worktree root
- [ ] `just test unit` and `go test -tags dev ./...` pass
- [ ] `just lint` and `BUILD_TAGS=dev just lint` pass
- [ ] Manual verification: create a worktree for a merged branch, add an uncommitted file, run `camp project prune` -- the worktree branch is listed as skipped with `dirty_worktree` reason
- [ ] Manual verification: same scenario with `--discard-dirty` -- the worktree is removed
- [ ] Manual verification: `camp fresh` with a dirty merged-branch worktree preserves the worktree

## Out of Scope

- Detached-worktree dirty check (already correct at `prune.go:302-323`; verified clean by the second pass).
- The `deleteRemoteBranches` function does not remove worktrees; no change needed there.
- GAP-8 consistency work across all commands is a sequence 10/11 item; this task adds `--discard-dirty` only to prune and fresh.