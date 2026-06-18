---
fest_type: task
fest_id: 03_workitem_create_atomicity.md
fest_name: workitem_create_atomicity
fest_parent: 09_app_contracts
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.750901-06:00
fest_updated: 2026-06-15T00:44:04.222555-06:00
fest_tracking: true
---


# Task: Make `camp workitem create` atomic (N-9)

## Objective

Ensure that a failed `camp workitem create` leaves no partial directory behind, so that immediate retry succeeds without requiring a manual `camp workitem adopt`.

## Requirements

- [x] The target directory is NOT created until after `deriveUniqueRef` succeeds; or, if the directory is created before that point, any failure after directory creation removes exactly what was created before returning the error
- [x] A test that induces a failure after directory creation confirms no directory is left on disk and that immediate retry succeeds
- [x] The adopt path is unaffected: a genuinely pre-existing non-empty directory still produces the "use adopt" error
- [x] The fix handles the `MkdirAll` path, the `deriveUniqueRef` path, and any path between them (`.workitem` write, yaml marshal)

## Implementation

### Background

`internal/commands/workitem/create.go` (verified at worktree bc2ad1f) at lines 77-89:

```go
target := filepath.Join(campaignRoot, parent, slug)
if _, err := os.Stat(target); err == nil {
    return camperrors.NewValidation("path",
        "target directory already exists: "+target+" — use `camp workitem adopt` to attach metadata to an existing dir", nil)
}
if err := os.MkdirAll(target, 0o755); err != nil {      // <-- directory created here
    return camperrors.Wrap(err, "create directory")
}

ref, err := deriveUniqueRef(ctx, campaignRoot, cfg, id)  // <-- full Discover; can fail
if err != nil {
    return err                                            // <-- leaves empty dir on disk
}
```

`deriveUniqueRef` runs a full `Discover` over the campaign, which hard-fails on any unreadable corner (e.g. a permission error on `festivals/active`). If it fails, the empty directory at `target` is left on disk. A subsequent `camp workitem create` with the same slug hits the `os.Stat` guard and exits with the "use adopt" error, making retry impossible without manual cleanup. This is exactly the partial-success-retry failure shape that FA0011 fixed in festival-app.

### Option A: derive before mkdir (preferred)

Move `deriveUniqueRef` and all other pre-creation logic BEFORE `os.MkdirAll`. This is cleaner because no cleanup is needed.

```go
// Derive ref BEFORE creating any filesystem state.
ref, err := deriveUniqueRef(ctx, campaignRoot, cfg, id)
if err != nil {
    return err
}
questID := resolveQuestIDForCreate(ctx, cmd, campaignRoot, questSelector)

// Now create the directory (ref is valid, so failure from here is a real I/O error).
target := filepath.Join(campaignRoot, parent, slug)
if _, err := os.Stat(target); err == nil {
    return camperrors.NewValidation("path",
        "target directory already exists: "+target+" — use `camp workitem adopt` to attach metadata to an existing dir", nil)
}
if err := os.MkdirAll(target, 0o755); err != nil {
    return camperrors.Wrap(err, "create directory")
}
```

Then the rest of the function (yaml marshal, atomicWriteFile, navigation cache invalidation, JSON output) follows. Check whether `resolveQuestIDForCreate` has any I/O that could fail; if so, move it before `MkdirAll` too.

### Option B: cleanup on failure (fallback if derive cannot move)

If there is a reason the derive must happen after mkdir (unlikely given the current code), add a deferred cleanup:

```go
if err := os.MkdirAll(target, 0o755); err != nil {
    return camperrors.Wrap(err, "create directory")
}
dirCreated := true
defer func() {
    if dirCreated {
        _ = os.Remove(target) // Remove only if empty; non-empty means adopt path
    }
}()

ref, err := deriveUniqueRef(ctx, campaignRoot, cfg, id)
if err != nil {
    return err // defer fires, removes target
}
// ... rest of create ...
dirCreated = false // success; suppress cleanup
```

Use `os.Remove` (not `os.RemoveAll`) so that if something else already wrote into the directory (race), the remove fails safely rather than destroying content.

Option A is preferred because it is simpler and has no race window.

### Adopt path preservation

The adopt validation (`os.Stat(target) == nil`) stays in place before `MkdirAll`. After the fix, an empty leftover from a previous partial run will satisfy `os.Stat` and trigger the adopt error. That is intentional: the fix prevents new leftovers; existing ones from runs before this fix still require manual adopt or removal. Document this in a comment.

### Tests

Add tests in `internal/commands/workitem/create_test.go` (or alongside existing create tests):

1. **Induced failure test**: create a fixture campaign, make `deriveUniqueRef` fail by arranging an unreadable directory in the discovery path (e.g. `chmod 000` on a subdirectory, then restore in defer). Call `runCreate`. Assert exit is non-zero AND the target directory does not exist on disk.

2. **Retry succeeds test**: use the same fixture in a healthy state, call `runCreate` twice with the same slug. The first call succeeds. Assert the second call fails with the "use adopt" message (slug already exists), NOT with a leftover-empty-dir message.

3. **Adopt still works test**: pre-create the target directory with some content, call `runCreate`. Assert the "use adopt" error is returned.

Note: `deriveUniqueRef` indirection via an injectable function is not required; a filesystem-level fixture (readable campaign with known slug) is sufficient.

## Done When

- [x] All requirements met
- [x] `deriveUniqueRef` (and `resolveQuestIDForCreate`) called before `os.MkdirAll` in `runCreate`
- [x] An induced failure after `MkdirAll` leaves no empty directory on disk
- [x] Immediate retry on a clean fixture succeeds
- [x] Adopt path still returns the correct error on a genuinely pre-existing directory
- [x] Both-profile gate passes
