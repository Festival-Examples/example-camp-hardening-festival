---
fest_type: task
fest_id: 03_workitem_list_read_only.md
fest_name: workitem_list_read_only
fest_parent: 08_argv_and_read_paths
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.731461-06:00
fest_updated: 2026-06-15T00:02:46.643464-06:00
fest_tracking: true
---


# Task: Remove priority pruning from workitem list (R18)

## Objective

Remove `priority.Prune` and `priority.SaveOrDelete` from the `camp workitem` list RunE so neither human nor `--json` mode writes to `.campaign/settings/workitems.json` during a read; move pruning to the explicit write paths that already own mutation: the priority command, TUI mutations, and `workitem doctor --fix`.

## Requirements

- [ ] `internal/commands/workitem/workitem.go:77-91` (verified): the `priority.Prune` / `priority.SaveOrDelete` block is removed from the list RunE; `priority.Load` and `priority.Apply` remain (they are read-only).
- [ ] `priority.WithLock` (introduced in sequence 06 task 04) is the sanctioned mutation door; when the pruning moves to the priority command's write path, it runs inside `WithLock`.
- [ ] `internal/commands/workitem/priority.go`: after a successful `Set` or `Clear`, prune stale entries using the full discovery set to keep the store tidy; this must use `WithLock`.
- [ ] `internal/workitem/tui/update.go:296-319` (TUI set/clear path): after any priority mutation, prune inside the same lock transaction.
- [ ] `internal/commands/workitem/doctor.go` `--fix` path: when `--fix` prunes priority entries, verify `priority.WithLock` is used (check: the current doctor code does not touch the priority store; if so, the fix is additive: add a pruning step after the links fix is committed and `knownIDs` is fresh).
- [ ] Tests: three named tests documented below.

## Implementation

### Background

The bug is in `internal/commands/workitem/workitem.go:77-91` (verified at worktree `bc2ad1f`):

```go
storePath := priority.StorePath(campaignRoot)
store, err := priority.Load(storePath)
if err != nil {
    return camperrors.Wrap(err, "loading priority store")
}
validKeys := make(map[string]bool, len(items))
for _, item := range items {
    validKeys[item.Key] = true
}
if priority.Prune(store, validKeys) {
    if err := priority.SaveOrDelete(storePath, store); err != nil {
        return camperrors.Wrap(err, "saving pruned priority store")
    }
}
```

This runs on every `camp workitem` invocation, including `camp workitem --json`. When a festival is mid-move between stage dirs (e.g., `festivals/active/` to `festivals/ready/`), the discovery temporarily misses it and `priority.Prune` deletes its priority entry permanently. The next `git status` shows a dirty `.campaign/settings/workitems.json`. festival-app polls `camp workitem --json` and triggers this path on every poll cycle.

The second finding is the lack of `WithLock`: even without the pruning, the current `Load` / `SaveOrDelete` pair in `priority.go:85-97` is an unlocked read-modify-write. Task 04 of sequence 06 adds `priority.WithLock`; this task removes the list-side write and wires the write paths through that lock.

### Step 1: Remove pruning from the list RunE

In `internal/commands/workitem/workitem.go`, replace lines 77-91 with a simpler block that only reads:

```go
storePath := priority.StorePath(campaignRoot)
store, err := priority.Load(storePath)
if err != nil {
    return camperrors.Wrap(err, "loading priority store")
}
```

The `validKeys` computation and the `Prune`/`SaveOrDelete` block are deleted entirely. `priority.Apply` and `wkitem.Sort` calls below are unchanged.

After this change the `priority` import may lose its `SaveOrDelete` and `Prune` usage; the `Prune` and `SaveOrDelete` symbols stay in the `priority` package for use by write paths.

### Step 2: Add pruning to the priority command write path

In `internal/commands/workitem/priority.go`, the `runPriority` function currently does a bare `Load`/`Set or Clear`/`SaveOrDelete` without a lock. After sequence 06 task 04 adds `priority.WithLock`, refactor to:

```go
func runPriority(ctx context.Context, cmd *cobra.Command, selectorArg, levelArg string, jsonOut bool) error {
    level, clear, err := parsePriorityLevel(levelArg)
    if err != nil {
        return err
    }

    cfg, root, err := config.LoadCampaignConfigFromCwd(ctx)
    if err != nil {
        return camperrors.Wrap(err, "not in a campaign directory")
    }
    resolver := paths.NewResolverFromConfig(root, cfg)

    wi, err := selector.Resolve(ctx, root, selectorArg, selector.ResolveOptions{})
    if err != nil {
        return err
    }

    storePath := priority.StorePath(root)
    return priority.WithLock(ctx, storePath, func(store *priority.Store) error {
        if clear {
            priority.Clear(store, wi.Key)
        } else {
            priority.Set(store, wi.Key, level)
        }

        // Prune orphaned entries now that we hold the lock and have a valid
        // discovery set. This is the only path that prunes.
        items, _ := wkitem.Discover(ctx, root, resolver)
        validKeys := make(map[string]bool, len(items))
        for _, item := range items {
            validKeys[item.Key] = true
        }
        priority.Prune(store, validKeys)

        if jsonOut {
            return emitPriorityJSON(cmd.OutOrStdout(), wi.Key, level, clear)
        }
        if clear {
            _, err := fmt.Fprintf(cmd.OutOrStdout(), "cleared priority: %s\n", wi.Key)
            return err
        }
        _, err = fmt.Fprintf(cmd.OutOrStdout(), "priority set: %s = %s\n", wi.Key, level)
        return err
    })
}
```

Note: `priority.WithLock` in the sequence 06 design takes a `func(*priority.Store) error` and handles Load/Save around it; adapt the signature to match whatever sequence 06 task 04 delivers. If sequence 06 has not landed yet when this task executes, implement `WithLock` inline using `fsutil.AcquireFileLock` directly and mark it as a cleanup target.

### Step 3: TUI mutation path

In `internal/workitem/tui/update.go:296-319` (verify exact lines at execution time), the TUI set/clear path must run inside the same `priority.WithLock` pattern. The TUI already has access to `storePath`; wrap the existing set/clear in a `WithLock` call that also prunes after the mutation.

### Step 4: Doctor --fix path

In `internal/commands/workitem/doctor.go`, the `--fix` branch (around line 109-121, verified) runs `links.WithLock` but does not touch the priority store. Add a priority prune step after the links fix completes and `knownIDs` is refreshed. This is additive and does not change the existing links-fix flow:

```go
// After links fix and knownIDs rescan:
if err := priority.WithLock(ctx, priority.StorePath(root), func(store *priority.Store) error {
    validPriorityKeys := make(map[string]bool, len(knownIDs))
    for id := range knownIDs {
        validPriorityKeys[id] = true
    }
    priority.Prune(store, validPriorityKeys)
    return nil
}); err != nil {
    // non-fatal: log to stderr and continue
    fmt.Fprintf(cmd.ErrOrStderr(), "warning: priority prune during fix: %v\n", err)
}
```

### Tests

**Test 1: list leaves store file byte-for-byte unchanged when discovery misses a key**

```go
func TestWorkitemList_NoPruneOnRead(t *testing.T) {
    dir := t.TempDir()
    // Seed a priority store with key "festival:festivals/active/demo"
    storePath := priority.StorePath(dir)
    store := priority.NewStore()
    priority.Set(store, "festival:festivals/active/demo", priority.High)
    priority.Save(storePath, store)
    originalData, _ := os.ReadFile(storePath)

    // Simulate list with an empty discovery (no festivals found, transient gap).
    // Call the pure apply+sort path directly without the prune block.
    // ... assertion: storePath content is byte-for-byte originalData after the call.

    afterData, _ := os.ReadFile(storePath)
    if !bytes.Equal(originalData, afterData) {
        t.Error("list mutated the priority store file")
    }
}
```

In practice, test the `outputJSON` path by constructing items that omit the priority key and verifying the file is unchanged.

**Test 2: doctor --fix prunes orphaned entries**

```go
func TestDoctorFix_PrunesOrphanPriority(t *testing.T) {
    // Seed a store with a key for a nonexistent workitem.
    // Run doctor --fix.
    // Assert the store no longer contains that key.
}
```

**Test 3: priority set still writes correctly**

Confirm the existing priority command tests (in `internal/commands/workitem/priority.go`'s test file) still pass after the RunE refactor; no test for "prunes nothing it should not" is needed beyond the existing valid-key scenario.

### Out of scope

- `jsoncontract` adoption for the `camp workitem` command itself is sequence 09/10; this task only removes the write side effect.
- The `--json=false` handling in `jsoncontract.RunE` is a verified-negative (C-3); no change needed.
- The TUI's `store` field caching is not addressed here; post-fix the TUI will load fresh on each mutation via `WithLock` rather than using its in-memory copy.

## Done When

- [ ] All requirements met
- [ ] `go test -count=1 -short ./internal/commands/workitem/... ./internal/workitem/priority/...` passes in both profiles
- [ ] `grep -n "priority.Prune\|priority.SaveOrDelete" internal/commands/workitem/workitem.go` returns zero matches
- [ ] Running `camp workitem --json` against a fixture with a known priority entry and an artificially empty discovery does not modify the store file (manual verification or test)