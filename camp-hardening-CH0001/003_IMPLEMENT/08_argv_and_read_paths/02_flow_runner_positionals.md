---
fest_type: task
fest_id: 02_flow_runner_positionals.md
fest_name: flow_runner_positionals
fest_parent: 08_argv_and_read_paths
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.731236-06:00
fest_updated: 2026-06-14T23:58:30.702134-06:00
fest_tracking: true
---


# Task: Fix flow runner extra-arg splicing (R5 flow runner part)

## Objective

Replace the `strings.Join`-into-`sh -c` extra-arg splicing in `internal/flow/runner.go` with a positional-arg pass-through (`sh -c '<cmd> "$@"' sh arg1 arg2 ...`) so extra args survive as distinct positionals and cannot be interpreted as shell code.

## Requirements

- [ ] `internal/flow/runner.go:32-37` (verified): the `command + " " + strings.Join(extraArgs, " ")` concatenation is removed; extra args are passed as positionals instead.
- [ ] The registered `f.Command` remains executed via `sh -c` because it is a full shell expression (the registry is user-controlled, like a justfile); only the arg splicing is the bug.
- [ ] An extra arg containing `$(...)`, `;`, `|`, or spaces is data, not code.
- [ ] When `extraArgs` is empty the behavior is unchanged: `sh -c f.Command` with no positionals.
- [ ] Context cancellation and exit code propagation are unchanged.
- [ ] Tests: a table-driven test using a fake flow registry command verifies the named scenarios.

## Implementation

### Background

The bug is in `internal/flow/runner.go:32-37` (verified at worktree `bc2ad1f`):

```go
command := f.Command
if len(extraArgs) > 0 {
    command = command + " " + strings.Join(extraArgs, " ")
}

cmd := exec.CommandContext(ctx, "sh", "-c", command)
```

When a user runs `camp flow run deploy -- "us east"`, this produces:

```
sh -c "my-deploy-script us east"
```

`us east` is word-split into two arguments. Worse, `camp flow run deploy -- "; rm -rf ."` executes the rm. The fix is the standard POSIX shell positional-passing idiom.

### Step 1: Update the `Run` method in `internal/flow/runner.go`

The full updated `Run` method:

```go
func (r *Runner) Run(ctx context.Context, f Flow, extraArgs []string) error {
    if err := ctx.Err(); err != nil {
        return fmt.Errorf("context cancelled: %w", err)
    }

    workDir := r.resolveWorkDir(f.WorkDir)

    // f.Command is a shell expression (user-controlled registry, like a justfile).
    // Pass extra args as positionals so they are never re-parsed as shell code:
    //   sh -c '<cmd> "$@"' sh arg1 arg2 ...
    // When there are no extra args, use the simpler sh -c '<cmd>' form.
    var shArgs []string
    if len(extraArgs) > 0 {
        shArgs = append([]string{"-c", f.Command + ` "$@"`, "sh"}, extraArgs...)
    } else {
        shArgs = []string{"-c", f.Command}
    }

    cmd := exec.CommandContext(ctx, "sh", shArgs...)
    cmd.Dir = workDir
    cmd.Env = r.mergeEnv(f.Env)
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        return fmt.Errorf("executing flow command: %w", err)
    }

    return nil
}
```

Note the argv structure for the extra-args case:

```
exec "sh" ["-c", "<cmd> \"$@\"", "sh", "arg1", "arg2", ...]
                                   ^-- $0 (argv[0] for the inline script)
```

In POSIX sh, `sh -c 'script' name args...` sets `$0 = name` and `$1, $2, ...` to the remaining args. `"$@"` expands them individually without re-splitting or re-quoting. The literal `"sh"` as argv[2] is the `$0` placeholder, conventional but arbitrary.

### Step 2: Remove the now-unused `strings` import

After the edit, `strings` is only used in `mergeEnv` via `strings.SplitN` and `strings.Join`. Verify the import is still needed; it is (`mergeEnv` uses it), so leave it.

### Step 3: Write tests

Create `internal/flow/runner_test.go` (or add to an existing test file). The test does not need a real shell invocation for the arg-splitting scenario; use a simple echo-to-file pattern:

```go
func TestRunner_ExtraArgPositionals(t *testing.T) {
    dir := t.TempDir()
    outFile := filepath.Join(dir, "out.txt")

    // The flow command writes its first arg to a file.
    // If arg-splicing is broken, "us east" would arrive as two separate args.
    cmd := fmt.Sprintf("printf '%%s\n' \"$1\" > %s", outFile)

    r := NewRunner(dir)
    f := Flow{Command: cmd, WorkDir: "."}

    if err := r.Run(context.Background(), f, []string{"us east"}); err != nil {
        t.Fatalf("Run: %v", err)
    }

    got, _ := os.ReadFile(outFile)
    if strings.TrimSpace(string(got)) != "us east" {
        t.Errorf("got %q, want %q", strings.TrimSpace(string(got)), "us east")
    }
}

func TestRunner_MetacharInArg_IsData(t *testing.T) {
    dir := t.TempDir()
    outFile := filepath.Join(dir, "out.txt")

    // The arg contains $(...) which must not be evaluated.
    cmd := fmt.Sprintf("printf '%%s\n' \"$1\" > %s", outFile)
    r := NewRunner(dir)
    f := Flow{Command: cmd, WorkDir: "."}

    if err := r.Run(context.Background(), f, []string{"$(echo injected)"}); err != nil {
        t.Fatalf("Run: %v", err)
    }

    got, _ := os.ReadFile(outFile)
    if strings.TrimSpace(string(got)) != "$(echo injected)" {
        t.Errorf("got %q, want literal string", strings.TrimSpace(string(got)))
    }
}

func TestRunner_EmptyExtraArgs_Unchanged(t *testing.T) {
    dir := t.TempDir()
    outFile := filepath.Join(dir, "out.txt")

    cmd := fmt.Sprintf("echo hello > %s", outFile)
    r := NewRunner(dir)
    f := Flow{Command: cmd, WorkDir: "."}

    if err := r.Run(context.Background(), f, nil); err != nil {
        t.Fatalf("Run: %v", err)
    }

    got, _ := os.ReadFile(outFile)
    if !strings.Contains(string(got), "hello") {
        t.Errorf("got %q, want 'hello'", string(got))
    }
}
```

Place these in `internal/flow/runner_test.go` following the host `t.TempDir()` convention (C-9). The tests use only `/bin/sh` built-ins and `printf`, which are available on the CI environment (macOS and Linux).

### Edge cases

- **Command containing `$@` already**: user flows that explicitly reference `$@` before this fix would have gotten empty expansion. After the fix, `$@` in the command string and `"$@"` appended are both present; the user's `$@` is now populated too. This is a net improvement; the behavior before was always wrong (empty positionals).
- **Extra args after `--` in camp flow run**: the cobra command for `camp flow run` must pass everything after `--` as `extraArgs`. Verify `internal/commands/flow/run.go` already splits on `--`; if not, that is a separate UX task (not part of this sequence).
- **Windows sh availability**: camp does not support Windows yet (project fact); no Windows guard needed.
- **Context cancellation mid-exec**: `exec.CommandContext` propagates cancellation via `cmd.Process.Kill()`; the wrapping `fmt.Errorf("context cancelled: %w", ...)` check at the top covers the pre-exec case. No change needed.

## Done When

- [ ] All requirements met
- [ ] `go test -count=1 -short ./internal/flow/...` passes in both profiles
- [ ] `camp flow run <entry> -- "arg with spaces"` delivers one positional (manual smoke against a test flow entry in a temp campaign)
- [ ] Grep confirms `strings.Join(extraArgs` no longer appears in `internal/flow/runner.go`