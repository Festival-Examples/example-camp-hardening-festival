---
fest_type: task
fest_id: 07_repair_failures_and_tty_guards.md
fest_name: repair_failures_and_tty_guards
fest_parent: 10_cli_consistency
fest_order: 7
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.77085-06:00
fest_updated: 2026-06-15T03:05:19.884415-06:00
fest_tracking: true
---


# Task: Repair Quiet Failures and TTY Guards

## Objective

Surface the three classes of silently swallowed failures (registry-save during init, fest-init cause, cleanup warnings), add IsTerminal guards to the four unguarded TUI commands, and ensure all four carry `agent_allowed: false` annotations so the manifest accurately reflects their interactive nature.

## Requirements

- [ ] `internal/scaffold/init.go:382-388` (registry-save during init): change `_ = config.SaveRegistry(ctx, reg)` to check the error and return it from `InitCampaign`. The user gets a clear failure rather than a silently unregistered campaign.
- [ ] `internal/scaffold/init.go:354-362` (fest init failure): distinguish "fest not installed" from "fest init failed" in `initFestivalsIfNeeded`. Currently both cases append the same "festivals/ (fest init failed - run manually)" skip message. After the fix: if `exec.LookPath("fest")` fails, append "festivals/ (fest not installed; run 'fest init' after installing)"; if `cmd.CombinedOutput()` fails, print `fmt.Fprintf(w, "Warning: fest init failed: %v\n", err)` to the writer and still append the skip entry.
- [ ] `internal/scaffold/repair.go:684` (swallowed RemoveAll): the `os.RemoveAll(m.Source)` call (currently guarded by an empty-check) should warn on stderr on failure: `fmt.Fprintf(os.Stderr, "camp repair warning: failed to remove legacy dir %s: %v\n", m.Source, err)`.
- [ ] `internal/project/add.go:261,266` (cleanup swallows warn): the two `_ = rmSection.Run()` and `_ = rmCached.Run()` calls should emit a warning on stderr if they fail, consistent with the repair pattern: `fmt.Fprintf(os.Stderr, "camp cleanup warning: %v\n", err)`.
- [ ] `cmd/camp/intent/explore.go`: add an `IsTerminal` check before the `tea.NewProgram` call (near line 121). If not a TTY, return `camperrors.Wrap(camperrors.ErrInvalidInput, "intent explore requires an interactive terminal")`. The command struct gains `agent_allowed: false` annotation.
- [ ] `cmd/camp/settings.go`: the command already has `agent_allowed: false` (verified at lines 31-35). Add the `IsTerminal` guard inside `runSettings` before the first `huh.NewForm` or `theme.RunForm` call (near line 59). Use the same `navtui.IsTerminal()` or `tui.IsTerminal()` check as the existing pattern.
- [ ] `cmd/camp/intent/crawl.go`: add an `IsTerminal` check early in its RunE body. Add `agent_allowed: false` to the command struct if not already present.
- [ ] `cmd/camp/dungeon/crawl.go`: add an `IsTerminal` check early in its RunE body. Add `agent_allowed: false` to the command struct if not already present.
- [ ] Coordinate with sequence 09 task 07 (manifest annotations): all four commands must end up annotated `agent_allowed: false` in both this task's structural fix and the manifest completeness sweep. Do not annotate them twice; if task 07 already added the annotation, verify it is correct and do not re-add.
- [ ] A single TTY-check helper is introduced in the appropriate package (e.g., `internal/ui/tty.go`) that checks both stdin AND stdout are terminals. All four commands use this helper. The existing separate stdin-check (`internal/nav/tui/keybindings.go:38`) and stdout-check (`internal/ui/theme/forms.go:18`) patterns are NOT consolidated globally (out of scope); only the four new guards use the shared helper.
- [ ] Tests: (a) a unit test verifying that calling `runSettings` with a non-TTY stderr/stdout (use `io.Discard` piped fds) returns the non-interactive error; (b) a unit test for init with a read-only registry path failing loudly (the registry-save failure should propagate to the caller).
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

Three classes of quiet failures are addressed here.

**Class 1: Registry-save during init.** `internal/scaffold/init.go:388` has `_ = config.SaveRegistry(ctx, reg)` inside the `if !opts.NoRegister && !opts.DryRun` block. A freshly initialized campaign silently fails to register; the user discovers it later when `camp switch` cannot find the campaign. This is the highest-severity quiet failure in this class (ERR-5 in the review).

**Class 2: Fest-init failure.** `cmd/camp/init/command.go:289` calls `initializeFestivals` with `festInitialized, _ = initializeFestivals(...)`, discarding the error. The underlying `internal/scaffold/init.go:354-362` catches the error internally but the message does not distinguish "fest binary not found" from "fest ran but failed". Users need to know which so they can take the right corrective action.

**Class 3: Cleanup warnings.** `internal/scaffold/repair.go:684` and `internal/project/add.go:261,266` discard errors from cleanup operations. These are not critical but should warn so users know why a re-run might behave unexpectedly.

**TTY guards:** Four TUI commands launch interactive interfaces with no terminal check. The good pattern at `cmd/camp/intent/add.go:159-164`:

```go
if !navtui.IsTerminal() {
    return camperrors.Wrap(camperrors.ErrInvalidInput, "title argument required in non-interactive mode\n       Usage: camp intent add <title> [flags]")
}
```

And at `cmd/camp/switch.go:86-88`:

```go
if !tui.IsTerminal() {
    return camperrors.Wrap(camperrors.ErrInvalidInput, "campaign name required in non-interactive mode...")
}
```

The four unguarded commands are: `intent explore` (launches `tea.NewProgram`), `settings` (launches `huh.NewForm` - already annotated, needs the runtime guard), `intent crawl`, and `dungeon crawl`.

Verify all cited line numbers against the worktree at `bc2ad1f` before editing.

### Verified file:line targets (as of review citations; verify before editing)

- `internal/scaffold/init.go:382-388` - `_ = config.SaveRegistry(ctx, reg)`
- `internal/scaffold/init.go:354-362` - `initFestivalsIfNeeded` failure handling
- `internal/scaffold/repair.go:684` - swallowed `os.RemoveAll(m.Source)` error
- `internal/project/add.go:261,266` - two `_ = rmSection.Run()` / `_ = rmCached.Run()`
- `cmd/camp/intent/explore.go:121` - `tea.NewProgram` without TTY check
- `cmd/camp/settings.go:43-69` - `runSettings` huh form without TTY check
- `cmd/camp/intent/crawl.go` - no IsTerminal reference
- `cmd/camp/dungeon/crawl.go` - no IsTerminal reference

### Step-by-step approach

**Step 1: Fix registry-save in `internal/scaffold/init.go`.**

Read the file near line 382. Find the block:

```go
if !opts.NoRegister && !opts.DryRun {
    reg, err := config.LoadRegistry(ctx)
    if err == nil {
        if err := reg.Register(campaignID, name, absDir, opts.Type); err == nil {
            _ = config.SaveRegistry(ctx, reg)
        }
    }
}
```

Change to:

```go
if !opts.NoRegister && !opts.DryRun {
    reg, err := config.LoadRegistry(ctx)
    if err != nil {
        return nil, camperrors.Wrap(err, "failed to load registry during init")
    }
    if err := reg.Register(campaignID, name, absDir, opts.Type); err != nil {
        return nil, camperrors.Wrap(err, "failed to register campaign")
    }
    if err := config.SaveRegistry(ctx, reg); err != nil {
        return nil, camperrors.Wrap(err, "failed to save registry during init")
    }
}
```

This makes the init function return an error if registration fails. Callers of `InitCampaign` (primarily `cmd/camp/init/command.go`) will propagate the error to the user. Verify that the function signature is `InitCampaign(...) (*InitResult, error)` before editing; if it returns only `error`, adjust accordingly.

**Step 2: Improve fest-init failure messaging.**

Read `internal/scaffold/init.go:initFestivalsIfNeeded` (currently around line 413). The function:
1. Returns `nil` silently if fest is not installed (via `exec.LookPath` failure).
2. Returns an error if fest runs but fails (via `cmd.CombinedOutput` failure).

The caller at line 354 (approximately) currently maps both cases to the same skip message. Change the function to distinguish the two cases by returning a sentinel or by having callers check. The simplest approach: keep the function returning `error`, but make the not-found case return a specific error type:

```go
festPath, err := exec.LookPath("fest")
if err != nil {
    return camperrors.Newf("fest not installed")
}
```

Then in the caller (in `init.go`), check:

```go
if err := initFestivalsIfNeeded(ctx, absDir); err != nil {
    if strings.Contains(err.Error(), "fest not installed") {
        result.Skipped = append(result.Skipped, "festivals/ (fest not installed; run 'fest init' after installing fest)")
    } else {
        result.Skipped = append(result.Skipped, fmt.Sprintf("festivals/ (fest init failed: %v; run 'fest init' manually)", err))
    }
}
```

A cleaner approach is to define a typed `ErrFestNotInstalled` sentinel. Either approach is acceptable; use the simpler one that avoids adding a new error type if the sentinel is only used here.

Also in `cmd/camp/init/command.go:289`, the `festInitialized, _ = initializeFestivals(...)` line discards errors. Read `initializeFestivals` to understand if it already wraps `initFestivalsIfNeeded` or if it is a separate path. After this fix, the error should be threaded from the scaffold function up through the command. Update `initializeFestivals` to return its error rather than discarding it.

**Step 3: Add cleanup warnings.**

In `internal/scaffold/repair.go`, find the `os.RemoveAll(m.Source)` call near line 684 (verify). Change from:

```go
os.RemoveAll(m.Source)
```

to:

```go
if err := os.RemoveAll(m.Source); err != nil {
    fmt.Fprintf(os.Stderr, "camp repair warning: failed to remove %s: %v\n", m.Source, err)
}
```

In `internal/project/add.go:261,266`, the two cleanup calls:

```go
_ = rmSection.Run()
// ...
_ = rmCached.Run()
```

Change each to:

```go
if err := rmSection.Run(); err != nil {
    fmt.Fprintf(os.Stderr, "camp warning: failed to clean .gitmodules entry: %v\n", err)
}
```

And:

```go
if err := rmCached.Run(); err != nil {
    fmt.Fprintf(os.Stderr, "camp warning: failed to unstage cached entry: %v\n", err)
}
```

**Step 4: Create the shared TTY helper.**

Create `internal/ui/tty.go`:

```go
package ui

import (
    "os"
    "golang.org/x/term"
)
```

Wait - check whether `golang.org/x/term` is already a dependency (check `go.mod`). If not, use the existing `navtui.IsTerminal()` function (which likely wraps a similar check). If `golang.org/x/term` is not present, look for the existing pattern: `internal/nav/tui/keybindings.go:38` checks stdin; `internal/ui/theme/forms.go:18` checks stdout. Create a helper in a package that both the cmd files and internal packages can import without import cycles. Check which packages `cmd/camp/intent/explore.go` and `cmd/camp/dungeon/crawl.go` already import.

The helper:

```go
// IsTTY reports whether both stdin and stdout are connected to a terminal.
func IsTTY() bool {
    return isTerminalFd(os.Stdin) && isTerminalFd(os.Stdout)
}
```

Where `isTerminalFd` wraps `term.IsTerminal(int(f.Fd()))` or equivalent.

If creating a new dependency would violate C-7, use the existing `navtui.IsTerminal()` (which appears to check stdin based on `keybindings.go:38`) as a temporary approach and document that a combined stdin+stdout check is a follow-up. The important thing is that non-TTY invocations return a clear error rather than a bubbletea panic.

**Step 5: Add IsTerminal guard to `intent/explore.go`.**

Read `cmd/camp/intent/explore.go`. Find the RunE body near line 121 where `tea.NewProgram` is called. Insert before it:

```go
if !navtui.IsTerminal() {
    return camperrors.Wrap(camperrors.ErrInvalidInput, "intent explore requires an interactive terminal")
}
```

Also update the command struct's `Annotations`:

```go
Annotations: map[string]string{
    "agent_allowed": "false",
    "agent_reason":  "Requires interactive terminal (TUI)",
    "interactive":   "true",
},
```

If `Annotations` does not exist on the command struct yet, add it.

**Step 6: Add IsTerminal guard to `settings.go`.**

Read `cmd/camp/settings.go`. The command already has `agent_allowed: false` in its Annotations (verified). Add the runtime guard at the top of `runSettings`, before the options/form construction:

```go
func runSettings(cmd *cobra.Command, args []string) error {
    if !tui.IsTerminal() {
        return camperrors.Wrap(camperrors.ErrInvalidInput, "settings requires an interactive terminal")
    }
    ctx := cmd.Context()
    // ...existing code...
}
```

The import for `tui` or `navtui` depends on which IsTerminal function is used; check existing imports in the file.

**Step 7: Add IsTerminal guard to `intent/crawl.go` and `dungeon/crawl.go`.**

Read both files. Find the RunE function body. Insert the guard early, before any TUI initialization. Add `agent_allowed: false` to the command struct if not already present.

For `intent/crawl.go`:

```go
if !navtui.IsTerminal() {
    return camperrors.Wrap(camperrors.ErrInvalidInput, "intent crawl requires an interactive terminal")
}
```

For `dungeon/crawl.go`:

```go
if !navtui.IsTerminal() {
    return camperrors.Wrap(camperrors.ErrInvalidInput, "dungeon crawl requires an interactive terminal")
}
```

**Step 8: Write tests.**

Create `internal/scaffold/init_registry_test.go` (host `t.TempDir()` convention):

```go
func TestInitCampaign_RegistrySaveFailure(t *testing.T) {
    // Create a temp dir with a read-only registry path
    // Attempt to init a campaign pointing at that registry
    // Verify the error is returned and is not nil
    // This may require setting CAMP_REGISTRY_PATH to a read-only location
}
```

Create `cmd/camp/settings_tty_test.go`:

```go
func TestSettings_NonTTY(t *testing.T) {
    // Call runSettings directly with a context
    // Verify it returns ErrInvalidInput (or a wrapped form) without panicking
}
```

Note: testing non-TTY behavior for commands that check stdin/stdout is tricky in unit tests because `os.Stdin` and `os.Stdout` may already be non-TTY in the test runner. Verify the `navtui.IsTerminal()` implementation to understand what it checks; a test that pipes stdin should reliably trigger the non-TTY path.

### Edge cases

**Registry-save failure propagation:** Making `config.SaveRegistry` errors fatal during `camp init` is a stronger behavior than the current "ignore". Verify that test fixtures do not create campaigns with intentionally dummy registry paths. The test for this behavior should set `CAMP_REGISTRY_PATH` to a read-only location using `t.Setenv`.

**Fest not installed vs failed:** If the codebase has tests for `camp init` that do not have `fest` on PATH, changing the skip message format may break those test assertions. Find and update any tests that assert the exact "fest init failed" string.

**Settings already annotated:** `cmd/camp/settings.go` already has `agent_allowed: false`. Do not add duplicate annotations. Read the file and confirm before editing.

**Sequence 09 task 07 coordination:** Task 09.07 sweeps all commands for annotation completeness. When that task runs, it will also encounter these four commands. The expectation is that this task adds the `agent_allowed: false` annotation to the three that lack it (explore, intent crawl, dungeon crawl); settings already has it. Task 09.07 then verifies and fills any remaining gaps. There should be no conflict.

### Out of scope

- `cmd/camp/switch.go:98` (`_ = config.SaveRegistry` on last-access update) is acceptable to degrade with a warning but is lower priority; not in scope for this task.
- `cmd/camp/project/add.go:248` (registry save error in the resolver path) is noted in ERR-5 but not in this task's INIT-2 scope; flag it for sequence 11 if needed.
- The "single TTY helper checking stdin AND stdout" as a CLI-wide refactor is noted as a Related-LOW in the findings; only the four guarded commands use the new helper here.

## Done When

- [ ] All requirements met
- [ ] `camp init` in a directory where the registry path is read-only exits with a clear error
- [ ] `camp init` without `fest` installed produces "fest not installed" in the output (not "fest init failed")
- [ ] `camp intent explore` when stdin is not a TTY returns a clear error message, not a bubbletea panic
- [ ] `camp settings` when stdin is not a TTY returns a clear error message
- [ ] `camp intent crawl` and `camp dungeon crawl` when non-TTY return clear errors
- [ ] All four commands have `agent_allowed: false` in their Annotations
- [ ] Unit tests pass: registry-save failure propagates, non-TTY settings call returns error
- [ ] Both-profile gate passes