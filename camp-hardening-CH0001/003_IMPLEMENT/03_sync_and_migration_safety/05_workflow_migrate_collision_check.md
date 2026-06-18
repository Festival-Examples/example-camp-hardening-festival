---
fest_type: task
fest_id: 05_workflow_migrate_collision_check.md
fest_name: workflow_migrate_collision_check
fest_parent: 03_sync_and_migration_safety
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.638271-06:00
fest_updated: 2026-06-14T20:21:41.760178-06:00
fest_tracking: true
---


# Task: Add destination-exists checks to MigrateV1ToV2 and collect per-item errors

**Finding:** R4 (WF-4)

## Objective

Add destination-exists checks to `MigrateV1ToV2` using the same pattern `service_move.go` already uses, collect per-item errors instead of aborting on the first failure, and ensure the v1 schema is preserved when the migration cannot complete cleanly.

## Requirements

- [ ] `MigrateV1ToV2` checks each destination with `resolveWorkflowItemPath` (already used in `service_move.go:46-49`) before any rename; if the destination exists, that item is recorded as a collision error.
- [ ] All items are checked first (or errors are collected per-item); the function does not silently clobber any destination under any code path.
- [ ] If any item errors (collision or rename failure), the function returns a non-nil error describing all failed items; the schema remains v1 (the `os.WriteFile` v2 schema write does NOT execute).
- [ ] A successful partial migration is not possible: either all items move and the schema is written to v2, or nothing is committed to disk and the schema stays v1 with a clear error.
- [ ] The collision-check helper is a small, named function with no-replace semantics, structured for D003 extraction in sequence 11.05.
- [ ] Note: the v2 schema write at `:221` (`os.WriteFile(s.schemaPath, data, 0644)`) is a plain `os.WriteFile`. Do NOT convert it to an atomic write here; that is covered by the sequence 06 atomic-write sweep. This task fixes only the collision logic.
- [ ] Unit tests: active/foo.md vs root foo.md collision returns an error and leaves both files untouched; active and ready sharing a name handled correctly; partial failure leaves schema at v1 with no partial moves.

## Implementation

### Background

`MigrateV1ToV2` at `internal/workflow/service_sync_migrate.go:152-229` (worktree bc2ad1f, verified):

```go
for _, statusDir := range []string{"active", "ready"} {
    dirPath := s.resolvePath(statusDir)
    if entries, err := os.ReadDir(dirPath); err == nil {
        for _, entry := range entries {
            name := entry.Name()
            if name == ".gitkeep" || name == "OBEY.md" {
                continue
            }
            src := filepath.Join(dirPath, name)
            dst := filepath.Join(s.root, name)   // line 182 in worktree

            if !dryRun {
                if err := os.Rename(src, dst); err != nil {  // line 185
                    return nil, camperrors.Wrapf(err, "moving %s to root", name)
                }
            }
            result.MovedItems = append(...)
        }
    }
}
```

`dst := filepath.Join(s.root, name)` puts every item from `active/` and `ready/` at the workflow root. If `active/foo.md` and root `foo.md` both exist, or if `active/foo.md` and `ready/foo.md` both exist, the second rename silently overwrites the first. Partial failure (first rename succeeds, second fails) aborts with the schema still at v1 but some items already moved.

The pattern to copy is `service_move.go:44-59`:

```go
if _, exists, err := resolveWorkflowItemPath(s.root, to, filepath.Base(itemPath)); err != nil {
    return nil, camperrors.Wrap(err, "failed to check destination")
} else if exists {
    return nil, camperrors.Wrap(ErrAlreadyExists, destPath)
}
```

### Step 1: Add a no-replace check helper

Add a small helper in `service_sync_migrate.go` (or near the `Move` helper in `service_move.go`):

```go
// checkMigrateDestination returns an error if dst already exists at the workflow root.
// Structured as a no-replace helper for D003 extraction in sequence 11.05.
func checkMigrateDestination(root, name string) error {
    dst := filepath.Join(root, name)
    if _, err := os.Stat(dst); err == nil {
        return camperrors.Wrapf(ErrAlreadyExists, "destination already exists: %s", dst)
    } else if !os.IsNotExist(err) {
        return camperrors.Wrapf(err, "checking destination: %s", dst)
    }
    return nil
}
```

### Step 2: Rewrite the item-moving loop in MigrateV1ToV2

Replace the inner loop body with:

```go
for _, statusDir := range []string{"active", "ready"} {
    dirPath := s.resolvePath(statusDir)
    entries, readErr := os.ReadDir(dirPath)
    if readErr != nil {
        if os.IsNotExist(readErr) {
            continue
        }
        return nil, camperrors.Wrapf(readErr, "reading %s", statusDir)
    }

    for _, entry := range entries {
        name := entry.Name()
        if name == ".gitkeep" || name == "OBEY.md" {
            continue
        }

        src := filepath.Join(dirPath, name)
        if err := checkMigrateDestination(s.root, name); err != nil {
            return nil, camperrors.Wrapf(err, "v1->v2 migration collision for %s/%s", statusDir, name)
        }

        if !dryRun {
            dst := filepath.Join(s.root, name)
            if err := os.Rename(src, dst); err != nil {
                return nil, camperrors.Wrapf(err, "moving %s/%s to root", statusDir, name)
            }
        }
        result.MovedItems = append(result.MovedItems,
            fmt.Sprintf("%s/%s -> ./%s", statusDir, name, name))
    }
}
```

The schema write block (`:215-225`) remains at the end of the function and is only reached when the loop above returns no error. No schema is written on any error path.

### Step 3: Unit tests

Add `internal/workflow/service_sync_migrate_test.go` (host `t.TempDir()`):

```go
func TestMigrateV1ToV2CollisionRejected(t *testing.T) {
    // Setup: workflow root with active/foo.md AND root foo.md
    root := t.TempDir()
    active := filepath.Join(root, "active")
    require.NoError(t, os.MkdirAll(active, 0755))
    require.NoError(t, os.WriteFile(filepath.Join(active, "foo.md"), []byte("active"), 0644))
    require.NoError(t, os.WriteFile(filepath.Join(root, "foo.md"), []byte("root"), 0644))
    writeV1Schema(t, root) // helper that writes a minimal v1 .workflow.yaml

    svc := newServiceForTest(root)
    _, err := svc.MigrateV1ToV2(context.Background(), false)
    if err == nil {
        t.Fatal("expected collision error, got nil")
    }

    // Both files must be in their original locations.
    assertFileContent(t, filepath.Join(active, "foo.md"), "active")
    assertFileContent(t, filepath.Join(root, "foo.md"), "root")

    // Schema must still be v1.
    schema := loadSchema(t, root)
    if schema.Version != 1 {
        t.Fatalf("schema was upgraded to v%d despite failure", schema.Version)
    }
}

func TestMigrateV1ToV2ActiveReadyNameCollision(t *testing.T) {
    // Setup: active/foo.md AND ready/foo.md both exist
    root := t.TempDir()
    active := filepath.Join(root, "active")
    ready := filepath.Join(root, "ready")
    require.NoError(t, os.MkdirAll(active, 0755))
    require.NoError(t, os.MkdirAll(ready, 0755))
    require.NoError(t, os.WriteFile(filepath.Join(active, "foo.md"), []byte("from-active"), 0644))
    require.NoError(t, os.WriteFile(filepath.Join(ready, "foo.md"), []byte("from-ready"), 0644))
    writeV1Schema(t, root)

    svc := newServiceForTest(root)
    _, err := svc.MigrateV1ToV2(context.Background(), false)
    // The first occurrence moves successfully; the second is a root collision.
    // The error must be non-nil and the schema must remain v1.
    if err == nil {
        t.Fatal("expected collision error for duplicate name across active/ready")
    }
    schema := loadSchema(t, root)
    if schema.Version != 1 {
        t.Fatalf("schema was upgraded to v%d despite failure", schema.Version)
    }
}
```

## Edge Cases

- **Both active/ and ready/ have the same filename**: The first rename succeeds and moves the item to the root; the second item encounters the now-existing root file and `checkMigrateDestination` fails. This is a real inconsistency in the v1 workflow that the user must resolve manually. The error message names the conflicting pair.
- **dryRun mode**: `checkMigrateDestination` runs even in dryRun so collisions are reported as dry-run failures, consistent with how repair shows errors without making changes.
- **v2 schema write not atomic**: This task explicitly leaves `:221` as `os.WriteFile`. The atomic-write conversion is tracked in sequence 06. The collision fix is independent of write atomicity.

## Out of Scope

- Atomic schema write (sequence 06).
- The `service_move.go` TOCTOU (sequence 11.05 shared primitive).
- Migrating the existing `service_move.go` destination check to the shared primitive (sequence 11.05).

## Done When

- [ ] `checkMigrateDestination` helper added with no-replace contract comment
- [ ] `MigrateV1ToV2` checks every destination before renaming; errors on collision
- [ ] Schema write is only reached when all items move successfully
- [ ] Unit tests: collision detected, schema stays v1, both file contents preserved
- [ ] Both-profile gate passes after these changes