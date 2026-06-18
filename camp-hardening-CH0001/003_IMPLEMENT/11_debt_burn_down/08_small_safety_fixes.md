---
fest_type: task
fest_id: 08_small_safety_fixes.md
fest_name: small_safety_fixes
fest_parent: 11_debt_burn_down
fest_order: 8
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.78892-06:00
fest_updated: 2026-06-15T10:36:26.873441-06:00
fest_tracking: true
---


# Task: Small Safety Fixes (N-20, N-22, N-27)

## Objective

Fix three low-severity but real safety issues: skills `--force` silently replacing user-owned symlinks, pins migration silently dropping valid pins under a symlinked campaign root, and buildutil clean running `rm -rf` globs in an unchecked working directory.

## Requirements

- [ ] `internal/skills/projection.go` with `--force`: unmanaged `StateBroken`/`StateMismatch` symlinks are treated like `StateNotALink` conflicts (not silently replaced) or the replaced target is printed before removal
- [ ] `internal/pins/pins.go MigrateAbsoluteToRelative` resolves the campaign ROOT via `filepath.EvalSymlinks` before computing `filepath.Rel`, so pins valid under a symlinked root are not silently dropped
- [ ] Dropped pins during migration are reported (not silent)
- [ ] `internal/buildutil/tasks/clean.go:46` shell glob `rm -rf` is guarded: refuses to run when the process cwd is outside the repository root, or uses explicit absolute paths instead of cwd-relative globs
- [ ] Tests for each fix
- [ ] Both-profile gate green

## Implementation

### Background

These three items are independent, low-complexity fixes. Each has a test and a clear target. The worktree is at `bc2ad1f`. Verify each target before editing.

### Fix 1: Skills `--force` unmanaged symlink protection (N-20)

Location: `internal/skills/projection.go:188-206`.

Verified in worktree: the `StateBroken` and `StateMismatch` cases under `force=true` call `os.Remove(destPath)` then `CreateSkillProjectionLink`. The `managed` check (`IsManagedSkillEntryLink`) is bypassed when `force=true` and the state is `StateBroken` or `StateMismatch`. A user-created symlink at the same path (e.g., a hand-crafted shortcut to a tool) is silently replaced.

Current code (approximate):
```go
case StateBroken, StateMismatch:
    managed, err := IsManagedSkillEntryLink(destPath, sourcePath, skillsDir)
    if err != nil {
        return summary, err
    }
    if !managed && !force {
        addConflict(&summary, slug)
        continue
    }
    if !dryRun {
        if err := os.Remove(destPath); err != nil && !os.IsNotExist(err) {
            return summary, camperrors.Wrapf(err, "remove broken skill link %q", slug)
        }
    }
    // ... CreateSkillProjectionLink
```

Fix: when `force=true` but the symlink is unmanaged, print the target being replaced to stdout/stderr before removing, so the user has visibility:

```go
case StateBroken, StateMismatch:
    managed, err := IsManagedSkillEntryLink(destPath, sourcePath, skillsDir)
    if err != nil {
        return summary, err
    }
    if !managed {
        if !force {
            addConflict(&summary, slug)
            continue
        }
        // Unmanaged symlink: print what is being replaced under --force
        target, readErr := os.Readlink(destPath)
        if readErr == nil {
            fmt.Fprintf(os.Stderr, "warning: replacing unmanaged symlink %q -> %q with skill link\n", destPath, target)
        } else {
            fmt.Fprintf(os.Stderr, "warning: replacing unmanaged broken symlink %q with skill link\n", destPath)
        }
    }
    if !dryRun {
        if err := os.Remove(destPath); err != nil && !os.IsNotExist(err) {
            return summary, camperrors.Wrapf(err, "remove broken skill link %q", slug)
        }
    }
    // ... CreateSkillProjectionLink
    summary.Replaced++
```

Test:
```go
func TestProjectSkillsForceUnmanagedSymlink(t *testing.T) {
    // Create a skills source dir with one skill
    // Create a destination dir with a hand-crafted symlink at the skill's expected path
    // Run ProjectSkills with force=true, dryRun=false
    // Verify the symlink was replaced (dest now points to source skill)
    // Verify a warning was printed to stderr
    // Verify no error was returned
}
```

### Fix 2: Pins migration under symlinked root (N-22)

Location: `internal/pins/pins.go:200-235` `MigrateAbsoluteToRelative`.

Verified in worktree: the function calls `filepath.EvalSymlinks` on each individual pin path (`:209-211`) but NOT on the `root` parameter. On macOS, if the campaign root was created as `/tmp/campaign` (a symlink to `/private/tmp/campaign`), and the pin was stored as `/private/tmp/campaign/projects/foo`, `filepath.Rel("/tmp/campaign", "/private/tmp/campaign/projects/foo")` fails the containment check (`rel` starts with `../`) and the pin is silently dropped.

Fix: resolve the root via `EvalSymlinks` at the start of the function:

```go
func (s *Store) MigrateAbsoluteToRelative(root string) bool {
    // Resolve the root to its canonical form to handle symlinked roots
    // (e.g., /tmp -> /private/tmp on macOS).
    if resolved, err := filepath.EvalSymlinks(root); err == nil {
        root = resolved
    }

    changed := false
    for i := len(s.pins) - 1; i >= 0; i-- {
        // ... existing per-pin logic ...
```

Also add reporting for dropped pins:

```go
} else {
    // External pin: cannot be made relative to this root.
    // Report before removing.
    fmt.Fprintf(os.Stderr, "warning: dropping pin %q: outside campaign root %q\n", s.pins[i].Path, root)
    s.pins = append(s.pins[:i], s.pins[i+1:]...)
    changed = true
}
```

Test:
```go
func TestMigrateAbsoluteToRelativeSymlinkedRoot(t *testing.T) {
    // Create a store with an absolute pin under a path that resolves to the
    // same location as root after EvalSymlinks (simulating /tmp vs /private/tmp).
    // Run MigrateAbsoluteToRelative with the unresolved root form.
    // Verify the pin is converted to relative rather than dropped.
    //
    // Implementation: use filepath.EvalSymlinks(t.TempDir()) to get the
    // resolved form; pass the unresolved form as root but store the pin
    // under the resolved path. On macOS this naturally exercises the fix.
    // On Linux where /tmp is not a symlink, use a manually created symlink
    // via os.Symlink in the test dir.
}
```

### Fix 3: Buildutil clean cwd guard (N-27)

Location: `internal/buildutil/tasks/clean.go:46`.

Verified in worktree: the `clean.go` function runs `rm -rf *.bak *.old *.backup *.wip *~` (and similar glob patterns) via `exec.Command("sh", "-c", fmt.Sprintf("rm -rf %s ...", pattern))` in the process's current working directory. If the process cwd is not the expected build output directory, these globs delete files from an unrelated directory.

This is a dev tool, not user-facing. The guard must prevent accidental damage if anything ever wires `buildutil.Clean` to a user-facing command.

Fix: add a repository root check before executing any glob-based removal:

```go
func (t *CleanTask) Run(opts CleanOptions) error {
    // Guard: refuse to run glob-based clean outside the known build output directory.
    // The build output directory is always within the repository root.
    cwd, err := os.Getwd()
    if err != nil {
        return fmt.Errorf("clean: cannot determine cwd: %w", err)
    }
    repoRoot, err := findRepoRoot(cwd)
    if err != nil {
        return fmt.Errorf("clean: cannot find repo root: %w", err)
    }
    if !strings.HasPrefix(cwd, repoRoot) {
        return fmt.Errorf("clean: cwd %q is outside repo root %q; refusing glob clean", cwd, repoRoot)
    }

    // ... existing clean logic
```

Alternatively, replace the glob-based `sh -c "rm -rf ..."` with explicit absolute path removal using the known build output directory:

```go
outDir := filepath.Join(repoRoot, "bin") // or wherever build output goes
patterns := []string{"*.bak", "*.old", "*.backup", "*.wip", "*~"}
// Use filepath.Glob on explicit directory, not cwd:
for _, pattern := range patterns {
    matches, _ := filepath.Glob(filepath.Join(outDir, pattern))
    for _, m := range matches {
        _ = os.Remove(m)
    }
}
```

The second approach (explicit absolute paths) is strictly safer and eliminates the shell entirely.

Test:
```go
func TestCleanRefusesOutsideRepoRoot(t *testing.T) {
    // Change to a temp directory that is NOT under any git repo
    // Attempt to run the clean task
    // Expect an error (not a successful silent glob execution)
}
```

Note: `internal/buildutil` is a dev tool (see ERR-9 informational note: "`exec.Command` without ctx is confined to `internal/buildutil` - a defensible exemption"). The `fmt.Errorf` in this file is also exempt from the task 01 ratchet since it falls in the non-production tool category. However, if the ratchet is configured to cover `internal/` broadly, add `internal/buildutil/tasks/clean.go` to the allowlist.

### Commit structure

Three independent changes; they can be in one commit or three:
- `fix(skills): warn before replacing unmanaged symlinks with --force (N-20)`
- `fix(pins): resolve symlinked root in MigrateAbsoluteToRelative (N-22)`
- `fix(buildutil): guard clean.go glob rm against non-repo cwd (N-27)`

## Done When

- [ ] `TestProjectSkillsForceUnmanagedSymlink` passes (unmanaged symlink replaced with warning, no error)
- [ ] `TestMigrateAbsoluteToRelativeSymlinkedRoot` passes (pin under symlinked root not dropped)
- [ ] `grep -n "rm -rf" internal/buildutil/tasks/clean.go` shows only repo-root-guarded or explicit-path invocations
- [ ] `TestCleanRefusesOutsideRepoRoot` passes
- [ ] Both-profile gate green