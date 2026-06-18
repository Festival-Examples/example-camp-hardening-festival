---
fest_type: task
fest_id: 04_alias_and_usage_fixes.md
fest_name: alias_and_usage_fixes
fest_parent: 10_cli_consistency
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.769719-06:00
fest_updated: 2026-06-15T02:36:46.056919-06:00
fest_tracking: true
---


# Task: Alias Collision and Usage Fixes

## Objective

Remove the `reg` alias from `register.go` so `camp reg` stops silently triggering a mutating command, set `SilenceUsage` globally on the root to stop runtime errors from dumping usage text, wire a `SetFlagErrorFunc` that re-enables usage display specifically for cobra flag parse errors with exit 2, and delete or wire the two dead persistent flags `--config` and `--verbose`.

## Requirements

- [ ] `cmd/camp/register.go:36`: `Aliases: []string{"reg"}` removed (or the slice becomes empty). The `registry` command at `cmd/camp/registry/root.go:10` retains its `Aliases: []string{"reg"}` unchanged.
- [ ] `cmd/camp/root.go:44`: `SilenceUsage: true` added to the root `cobra.Command` struct literal (alongside the existing `SilenceErrors: true`).
- [ ] Per-command `SilenceUsage: true` patches removed from: `cmd/camp/id.go:17`, `cmd/camp/root_path.go:26`, `cmd/camp/skills/status.go:27`, `internal/commands/workflow/doctor.go:72`, `internal/commands/workitem/doctor.go:75`. Also remove the runtime flip `cmd/camp/pull.go:75` which sets `cmd.SilenceUsage = true` inside RunE.
- [ ] A `SetFlagErrorFunc` is installed on `rootCmd` that re-enables usage output for cobra flag parse errors and returns a `camperrors.CommandError` with exit code 2 (per the table from task 02). Pattern: cobra calls `FlagErrorFunc` before RunE when a flag fails to parse; the hook should re-enable usage on the command, then return a typed CommandError that the main mapper converts to exit 2.
- [ ] `cmd/camp/root.go:192`: the `--config` flag is either deleted (if `cfgFile` is confirmed unused) or wired to actually load the config file it claims to configure. Confirmed unused by the finding (grep shows only declaration plus bind); delete both the flag declaration and the `cfgFile` variable.
- [ ] `cmd/camp/root.go:194`: the `--verbose` flag is either deleted or made functional. The review confirmed that only `flow sync` reads the persistent `--verbose` flag (`internal/commands/flow/sync.go:59`). Since `flow` is dev-only and the four local `--verbose` flags in sync, clone, doctor, and leverage each define their own local binding, the global persistent `--verbose` is effectively non-functional for all stable commands. Delete the persistent flag and the `verbose` package variable; the local `--verbose` flags in sync, clone, doctor, and leverage are correct and remain.
- [ ] A test in `cmd/camp/root_test.go` or a new file verifies: (a) no two root children share the same alias; (b) a runtime error (e.g., `camp intent show does-not-exist`) does not print usage text; (c) a bad flag (`camp intent show --bad-flag`) does print usage text.
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

Two root children both claim the `reg` alias: `register` (at `cmd/camp/register.go:36`) and `registry` (at `cmd/camp/registry/root.go:10`). Verified live: `camp reg` fires `register`, which immediately writes to the registry. The conflict is a fragile cobra registration-order dependency.

`SilenceErrors: true` is set on root (suppresses re-printing the error), but `SilenceUsage` is not set. Every runtime error therefore dumps ~12 lines of usage text before the `Error:` line, making runtime failures indistinguishable from bad-flag errors. The codebase already fixed this piecemeal on some commands; this task applies the root-level fix.

The flag parse case is different from runtime errors: when a user types `--bad-flag`, cobra should show usage because that is a user mistake correctable by reading the flags. The `SetFlagErrorFunc` hook handles this by setting `cmd.SilenceUsage = false` (re-enabling it) for parse errors and returning a typed error for exit 2 mapping.

Verify all cited line numbers against the worktree at `bc2ad1f` before editing.

### Verified file:line targets (as of review citations; verify before editing)

- `cmd/camp/register.go:36` - `Aliases: []string{"reg"}`
- `cmd/camp/registry/root.go:10` - `Aliases: []string{"reg"}` (keep this one)
- `cmd/camp/root.go:44` - root `cobra.Command` struct literal
- `cmd/camp/root.go:192-194` - `--config` and `--verbose` flag declarations
- `cmd/camp/id.go:17` - per-command `SilenceUsage: true`
- `cmd/camp/root_path.go:26` - per-command `SilenceUsage: true`
- `cmd/camp/skills/status.go:27` - per-command `SilenceUsage: true`
- `internal/commands/workflow/doctor.go:72` - per-command `SilenceUsage: true`
- `internal/commands/workitem/doctor.go:75` - per-command `SilenceUsage: true`
- `cmd/camp/pull.go:75` - runtime `cmd.SilenceUsage = true` inside RunE

### Step-by-step approach

**Step 1: Remove the `reg` alias from `register.go`.**

Read `cmd/camp/register.go`. Find the `Aliases` field in the command struct:

```go
Aliases: []string{"reg"},
```

Delete this line or change it to `Aliases: nil`. The `register` command is still accessible via its full name `camp register`. Leave `cmd/camp/registry/root.go` unchanged.

**Step 2: Set `SilenceUsage` on the root.**

Read `cmd/camp/root.go`. Find the `rootCmd` variable declaration with its `cobra.Command` struct. Add `SilenceUsage: true` next to the existing `SilenceErrors: true`:

```go
var rootCmd = &cobra.Command{
    Use:          "camp",
    Short:        "...",
    SilenceErrors: true,
    SilenceUsage:  true,
    // ...
}
```

**Step 3: Remove per-command `SilenceUsage: true` patches.**

Read each of the five files listed and delete the `SilenceUsage: true` line from the command struct literal. Each file has it in a `cobra.Command` struct, e.g.:

```go
var idCmd = &cobra.Command{
    Use:          "id",
    SilenceUsage: true,  // delete this line
    RunE:         runID,
}
```

For `cmd/camp/pull.go:75`, read the RunE function body and remove the `cmd.SilenceUsage = true` statement; it was a runtime workaround no longer needed.

**Step 4: Install `SetFlagErrorFunc` on root.**

In `cmd/camp/root.go`, inside the `init()` function or wherever root-level setup happens (look for where `rootCmd.PersistentFlags()` is called), add:

```go
rootCmd.SetFlagErrorFunc(func(cmd *cobra.Command, err error) error {
    cmd.SilenceUsage = false
    return camperrors.NewCommand(cmd.CommandPath(), 2, "", err)
})
```

This hook fires only when cobra fails to parse a flag (unknown flag, invalid value, etc.). It re-enables usage display for that specific command and returns a `CommandError` with exit 2, which `main.go`'s mapper converts to `os.Exit(2)`.

**Step 5: Delete `--config` and `--verbose` flags.**

Read `cmd/camp/root.go` near line 192. The declarations:

```go
rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file ...")
rootCmd.PersistentFlags().BoolVar(&verbose, "verbose", false, "enable verbose output")
```

Delete both lines. Then find the `cfgFile` and `verbose` variable declarations (likely in a `var` block near the top of `root.go`) and delete those too. Run `grep -n "cfgFile\|verbose\b" cmd/camp/root.go` to confirm all sites.

For `verbose`: the local `--verbose` flags in `cmd/camp/sync.go:77`, `cmd/camp/clone.go:105`, `cmd/camp/doctor.go:70`, and `cmd/camp/leverage/main_command.go:21` bind their own local variables (not the deleted global). They are correct and should remain unchanged.

**Step 6: Write the tests.**

In `cmd/camp/root_test.go` (or a new `cmd/camp/alias_test.go` file, host `t.TempDir()` convention), write:

```go
func TestNoAliasCollisions(t *testing.T) {
    seen := map[string]string{}
    for _, cmd := range rootCmd.Commands() {
        for _, alias := range cmd.Aliases {
            if prev, ok := seen[alias]; ok {
                t.Errorf("alias %q is used by both %q and %q", alias, prev, cmd.Name())
            }
            seen[alias] = cmd.Name()
        }
    }
}

func TestRuntimeErrorNoUsageDump(t *testing.T) {
    // Execute a command that will error at runtime (not flag parse)
    // Verify stdout/stderr does not contain usage text like "Usage:" or "Flags:"
    // (use a pipe-captured stderr)
}

func TestFlagParseErrorShowsUsage(t *testing.T) {
    // Execute with a non-existent flag
    // Verify stderr contains "Usage:" section and exit code is 2
}
```

For the runtime and flag-parse tests, capture `rootCmd.OutOrStderr()` by calling `rootCmd.SetErr(buf)` before the test execution, then checking the captured output.

### Edge cases

**`internal/commands/workflow/doctor.go` and `internal/commands/workitem/doctor.go`:** These are in `internal/`, not `cmd/`. Read both files carefully; the `SilenceUsage: true` is on the inner subcommand, which inherits root's `SilenceUsage` anyway once root sets it. Removing the per-command setting is safe.

**`cmd/camp/pull.go:75` runtime flip:** Some commands use `cmd.SilenceUsage = true` at the point where an error is about to be returned to suppress usage after a runtime failure. After root sets `SilenceUsage: true` globally, the explicit runtime flip is a no-op. Remove it.

**`SetFlagErrorFunc` and jsoncontract:** Task 01 wraps commands with `jsoncontract.RunE`. The `jsoncontract` package has its own `FlagErrorFunc` helper for JSON mode. After task 01 lands, the root `SetFlagErrorFunc` set here should be compatible: if a command uses `jsoncontract.FlagErrorFunc`, that command's `SetFlagErrorFunc` overrides the root's for that command. The root-level hook is a fallback for commands not wrapped with `jsoncontract`. Verify this does not double-wrap by checking whether any `jsoncontract.FlagErrorFunc` calls are already wired for the wrapped commands.

### Out of scope

- UX-2/3/5/10 help text polish (rename `promote`, `wt` alias, etc.) is sequence 12.02.
- The `--verbose` flag on `flow sync` (`internal/commands/flow/sync.go:59`) reads the persistent flag; after the global is deleted, `flow sync`'s verbose behavior breaks. Since `flow` is dev-only and this is a behavior change in a maturing surface, this is acceptable. Add a note in `docs/exit-codes.md` or a comment in `flow/sync.go` that the persistent global was removed and the local flag should be used instead. Alternatively, add a local `--verbose` flag to `flow sync` before deleting the global. Check whether `flow sync` already has one.

## Done When

- [ ] All requirements met
- [ ] `camp reg` routes to `registry` (reads, does not write)
- [ ] `camp intent show does-not-exist-xyz` prints only `Error: ...` with no usage block before it
- [ ] `camp intent show --bad-flag` prints usage text and exits 2
- [ ] `grep 'SilenceUsage: true' cmd/camp/id.go cmd/camp/root_path.go cmd/camp/skills/status.go cmd/camp/pull.go` returns no results
- [ ] `grep 'cfgFile\|--config' cmd/camp/root.go` returns no results
- [ ] `TestNoAliasCollisions` passes in both profiles
- [ ] Both-profile gate passes