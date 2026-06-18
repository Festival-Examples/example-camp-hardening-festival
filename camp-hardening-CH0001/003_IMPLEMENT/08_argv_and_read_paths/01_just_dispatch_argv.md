---
fest_type: task
fest_id: 01_just_dispatch_argv.md
fest_name: just_dispatch_argv
fest_parent: 08_argv_and_read_paths
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.730922-06:00
fest_updated: 2026-06-14T23:55:05.793197-06:00
fest_tracking: true
---


# Task: Fix argv flattening in camp run just-dispatch (R5/RUN-1)

## Objective

Replace the `strings.Join`-into-`sh -c` path used by project just-dispatch with a direct `exec.CommandContext(ctx, "just", args...)` call so arguments containing spaces, quotes, shell metacharacters, and leading dashes survive byte-for-byte through to just.

## Requirements

- [ ] `cmd/camp/cmdutil/command.go`: the existing `ExecuteCommand(ctx, "just", ...)` call path must exec `just` directly instead of joining args into `sh -c`; the raw-command form (all non-project callers) continues to use `sh -c` and its shell semantics are preserved and documented.
- [ ] `cmd/camp/run.go:183`: the project just-dispatch branch calls the new direct-exec helper (not the sh -c `ExecuteCommand`) with `cmd.Dir = projectDir` and `CAMPAIGN_ROOT` in the env.
- [ ] `cmd/camp/run.go:161,192`: the two remaining `strings.Join` flattenings on the shortcut-based and root-fallback paths are acceptable because they ARE raw-command form and use `sh -c`; verify they are correct and document them.
- [ ] Exit code propagation: `camperrors.NewCommand(fullCmd, exitErr.ExitCode(), ...)` must remain intact for the direct-exec path so `main.go`'s `os.Exit(cmdErr.ExitCode)` chain still works.
- [ ] Tests: a test just recipe (or fake executable on PATH placed in `t.TempDir`) proves: (a) an arg with spaces survives as one argument, (b) an arg with `$VAR` is not shell-expanded, (c) an arg with a semicolon is data not a command separator, (d) an arg with a glob `*` is not expanded, (e) an arg with leading `--` and a `-flag` form is passed through, (f) exit codes non-zero from just still propagate.

## Implementation

### Background

The bug is in `cmd/camp/cmdutil/command.go:22-27` (verified at worktree branch `bc2ad1f`):

```go
fullCmd := cmdStr
if len(extraArgs) > 0 {
    fullCmd = fmt.Sprintf("%s %s", cmdStr, strings.Join(extraArgs, " "))
}
cmd := exec.CommandContext(ctx, "sh", "-c", fullCmd)
```

When a caller passes `"just"` as `cmdStr` and `["commit", "-m", "fix: two words"]` as `extraArgs`, this produces `sh -c "just commit -m fix: two words"`, splitting the message across `fix:`, `two`, and `words`. This bites the `cr` alias on every commit with a multi-word message.

The fix requires splitting `ExecuteCommand` into two helpers or adding a parallel direct-exec path that skips the sh wrapper.

### Step 1: Add a direct-exec helper in `cmd/camp/cmdutil/command.go`

Add a new function alongside `ExecuteCommand`. The existing function stays for all raw-command callers.

```go
// ExecuteDirect executes the named binary with argv directly (no shell).
// Use this for project just-dispatch where argv must survive byte-exact.
// campaignRoot is threaded by callers to set CAMPAIGN_ROOT.
func ExecuteDirect(ctx context.Context, binary string, args []string, workDir, campaignRoot string) error {
    if ctx.Err() != nil {
        return ctx.Err()
    }

    argv := append([]string{binary}, args...)
    fullCmd := strings.Join(argv, " ") // for error messages only

    cmd := exec.CommandContext(ctx, binary, args...)
    cmd.Dir = workDir
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.Stdin = os.Stdin
    cmd.Env = os.Environ()
    if campaignRoot != "" {
        cmd.Env = append(cmd.Env, campaign.EnvCampaignRoot+"="+campaignRoot)
    }

    if err := cmd.Run(); err != nil {
        if exitErr, ok := err.(*exec.ExitError); ok {
            return camperrors.NewCommand(fullCmd, exitErr.ExitCode(), "", exitErr)
        }
        return camperrors.Wrap(err, "failed to execute command")
    }
    return nil
}
```

The `strings` import is already present. No new imports needed.

### Step 2: Update the project just-dispatch branch in `cmd/camp/run.go`

In `runRun`, at line 182-184 (verified):

```go
// Before (broken):
if projectDir, ok := isProjectCtx(ctx, root, commandArgs[0]); ok {
    return cmdutil.ExecuteCommand(ctx, "just", projectDir, root, commandArgs[1:])
}

// After (fixed):
if projectDir, ok := isProjectCtx(ctx, root, commandArgs[0]); ok {
    return cmdutil.ExecuteDirect(ctx, "just", commandArgs[1:], projectDir, root)
}
```

The rest of `runRun` (the shortcut path at line 161 and the root-fallback at line 192) already uses `sh -c` correctly for raw-command form. Leave them untouched.

### Step 3: Annotate the sh -c paths in `run.go`

Add a one-line comment to the raw-command `ExecuteCommand` calls (lines 161 and 192) to document the intentional shell semantics:

```go
// Raw-command form: shell interprets the joined args; shell metacharacters are intentional.
return cmdutil.ExecuteCommand(ctx, fullCmd, workDir, root, nil)
```

Similarly in `ExecuteCommand`'s body, add a comment above the `sh -c` block:

```go
// Raw-command form: cmdStr is a shell expression; sh -c is the intended execution model.
// For argv-safe dispatch use ExecuteDirect.
```

### Step 4: Write tests

Create `cmd/camp/cmdutil/command_test.go` (or add to an existing `_test.go` in the same package). Use a fake executable on `PATH`:

```go
func TestExecuteDirect_ArgSurvival(t *testing.T) {
    // Build a tiny helper that prints each arg on its own line.
    dir := t.TempDir()
    helperSrc := filepath.Join(dir, "printargs.go")
    // Write a Go program that prints os.Args[1:], one per line.
    os.WriteFile(helperSrc, []byte(`package main
import ("fmt"; "os")
func main() { for _, a := range os.Args[1:] { fmt.Println(a) } }`), 0644)
    helperBin := filepath.Join(dir, "printargs")
    exec.Command("go", "build", "-o", helperBin, helperSrc).Run()

    cases := []struct {
        name string
        args []string
        want []string
    }{
        {"spaces", []string{"fix: two words"}, []string{"fix: two words"}},
        {"dollar var", []string{"$HOME"}, []string{"$HOME"}},
        {"semicolon", []string{"a;b"}, []string{"a;b"}},
        {"glob", []string{"*.go"}, []string{"*.go"}},
        {"leading dash", []string{"--flag", "-v"}, []string{"--flag", "-v"}},
        {"double quote in arg", []string{`say "hi"`}, []string{`say "hi"`}},
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            var buf bytes.Buffer
            // Temporarily capture stdout via a pipe or use a custom helper.
            // Simplest: write a helper binary that appends to a file.
            outFile := filepath.Join(dir, tc.name+".out")
            // Use a wrapper that redirects stdout to outFile for assertion.
            // ... (see full harness pattern below)
        })
    }
}
```

A simpler test harness avoids pipe complexity: write the fake binary to a file, set up a PATH that includes `dir`, call `ExecuteDirect` with `binary = "printargs"`, and assert the output file contents.

The acceptance scenario documented in the plan: `cr just recipe -m "fix: two words"` routes through `isProjectCtx` -> `ExecuteDirect` -> `just` receives `["recipe", "-m", "fix: two words"]` as three argv entries.

### Edge cases

- **Flag parsing of leading-dash args after `--`**: just receives them as raw strings; just's own `--` pass-through is just's responsibility, not camp's.
- **A project named `just`**: `isProjectCtx` resolves by submodule name; a project literally called `just` would route here. No change needed; it was already the case before.
- **`cr` invoked from outside a campaign**: `campaign.DetectCached` returns an error, which propagates before dispatch. No change.
- **`ExecuteCommand` with `cmdStr = "just"` from non-run callers**: search for other callers of `ExecuteCommand` passing `"just"` as the command. At the verified worktree there are none outside `run.go:183`; the fix is local.

## Done When

- [ ] All requirements met
- [ ] `go test -count=1 -short ./cmd/camp/cmdutil/...` passes in both profiles
- [ ] Manual spot check: `camp run <project> <recipe> -m "message with spaces"` delivers the message intact (visible in just's output)
- [ ] `grep "ExecuteCommand.*just" cmd/camp/run.go` returns zero matches (the dispatch path no longer calls it)