---
fest_type: task
fest_id: 02_exit_code_table.md
fest_name: exit_code_table
fest_parent: 10_cli_consistency
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.769281-06:00
fest_updated: 2026-06-15T02:18:57.965213-06:00
fest_tracking: true
---


# Task: Exit Code Table and os.Exit Cleanup

## Objective

Define a single CLI-wide exit code table, align the three divergent command-specific tables (doctor, sync, clone) to that standard, and replace every `os.Exit` call inside `RunE` functions with typed `camperrors.CommandError` so the exit code flows through `cmd/camp/main.go`'s existing mapper without bypassing deferred cleanup.

## Requirements

- [ ] A CLI-wide exit code table is documented in `docs/exit-codes.md` (new file) with entries: 0=success, 1=runtime failure (command ran but something went wrong), 2=usage error (bad flags or args), 3+=command-specific partial states documented per command.
- [ ] `internal/doctor/options.go:141-148`: constants remapped so warnings and failures both map to 1; `ExitPartialFix` becomes 3; `ExitInvalidArgs` (if it exists) becomes 2.
- [ ] `internal/sync/options.go:181-188`: `ExitPreflightFailed`=1, `ExitSyncFailed`=1, `ExitValidationFailed`=3, `ExitInvalidArgs`=2.
- [ ] `internal/clone/options.go:248-255`: `ExitCloneFailed`=1, `ExitPartialSuccess`=3, `ExitValidationFailed`=3, `ExitInvalidArgs`=2.
- [ ] `cmd/camp/sync.go:144-146`: both `os.Exit` calls replaced by returning `camperrors.NewCommand("camp sync", <code>, "", nil/err)`.
- [ ] `cmd/camp/doctor.go` (exitDoctorWithCode, ~lines 221-232): `exitDoctorWithCode` changed from a void function calling `os.Exit` to an `error`-returning function; its three `os.Exit` calls replaced by `return camperrors.NewCommand(...)`.
- [ ] `cmd/camp/clone.go`: all five `os.Exit` sites (review citations 153, 161, 311, 317, 321; verify against worktree) replaced by `return camperrors.NewCommand(...)`.
- [ ] `cmd/camp/main.go` exit-code mapper at lines 11-21 is unchanged; it already handles `camperrors.CommandError.ExitCode`.
- [ ] Table-driven exit-code tests assert that `Execute()` returns a `camperrors.CommandError` with the expected exit code for doctor/sync/clone failure modes; no test calls `os.Exit` or uses `os/exec` to subprocess.
- [ ] camp is not publicly released; the old per-command codes (doctor 2=failures, clone 2=partial) are breaking changes that are explicitly acceptable and noted in `docs/exit-codes.md`.
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

Three packages define their own exit code tables today with incompatible meanings:

- `internal/doctor/options.go:142-148`: 1=warnings, 2=failures, 3=partial fix.
- `internal/sync/options.go:181-188`: 1=preflight failed, 2=sync failed, 3=validation failed.
- `internal/clone/options.go:248-255`: 1=clone failed, 2=partial SUCCESS, 3=validation failed.

Exit 2 means "hard failure" for sync, "failure" for doctor, and "mostly worked" for clone. A wrapper script cannot write a generic handler. The standard Unix convention and what `cmd/camp/main.go:20` already does for every other command is: 0=ok, 1=any failure, 2=usage error.

The `os.Exit` calls inside RunE bypass cobra's `Execute()` return value, deferred cleanup, and any test harness calling `Execute()` directly. `camperrors.CommandError` and the mapper at `main.go:14-17` already exist for this; the three commands just did not use them.

Verify all cited line numbers against the worktree at `bc2ad1f` before editing each file.

### Verified file:line targets (review citations; verify before editing)

- `internal/doctor/options.go:141-148` - exit code constants
- `internal/sync/options.go:177-188` - exit code constants (line numbers for the const block)
- `internal/clone/options.go:244-255` - exit code constants
- `cmd/camp/sync.go:144-146` - two `os.Exit` calls
- `cmd/camp/doctor.go:221-232` - `exitDoctorWithCode` with three `os.Exit` calls
- `cmd/camp/clone.go:153,161,311,317,321` - five `os.Exit` sites

### Step-by-step approach

**Step 1: Write `docs/exit-codes.md`.**

Create the file with the canonical table, rationale for collapsing doctor's 1/2 split (both signal "something is wrong, fix it"), the partial-state entries, and a migration note that the old per-command codes changed during CH0001.

**Step 2: Remap the three constants blocks.**

In `internal/doctor/options.go`, read the current block. If `ExitWarnings=1`, `ExitFailures=2`, `ExitPartialFix=3`, change to:

```go
const (
    ExitWarnings   = 1
    ExitFailures   = 1
    ExitPartialFix = 3
)
```

The duplicate value 1 is intentional and valid Go. If callers distinguish them by constant name (grep `doctor.ExitWarnings` and `doctor.ExitFailures` in `cmd/`), they still compile correctly because the value is the same; the constant names become aliases for 1.

In `internal/sync/options.go`, change to:

```go
const (
    ExitSuccess          = 0
    ExitPreflightFailed  = 1
    ExitSyncFailed       = 1
    ExitValidationFailed = 3
    ExitInvalidArgs      = 2
)
```

In `internal/clone/options.go`, change to:

```go
const (
    ExitSuccess          = 0
    ExitCloneFailed      = 1
    ExitPartialSuccess   = 3
    ExitValidationFailed = 3
    ExitInvalidArgs      = 2
)
```

**Step 3: Replace `os.Exit` in `cmd/camp/sync.go`.**

Read the file near line 144. The current pattern:

```go
if !result.Success {
    if !result.PreflightPassed && !syncOpts.force {
        os.Exit(sync.ExitPreflightFailed)
    }
    os.Exit(sync.ExitSyncFailed)
}
return nil
```

Replace with:

```go
if !result.Success {
    if !result.PreflightPassed && !syncOpts.force {
        return camperrors.NewCommand("camp sync", sync.ExitPreflightFailed, "", nil)
    }
    return camperrors.NewCommand("camp sync", sync.ExitSyncFailed, "", nil)
}
return nil
```

Remove the `"os"` import from `cmd/camp/sync.go` if it is no longer used after this change (check for other `os.` calls in the file first).

**Step 4: Refactor `exitDoctorWithCode` in `cmd/camp/doctor.go`.**

Read the current `exitDoctorWithCode` function. It currently calls `os.Exit` and returns nothing (or returns `error` for exit 0 only). Change its signature to return `error` and replace the exits:

```go
func exitDoctorWithCode(result *doctor.DoctorResult) error {
    if result.Failed > 0 {
        return camperrors.NewCommand("camp doctor", doctor.ExitFailures, "", nil)
    }
    if result.Warned > 0 {
        return camperrors.NewCommand("camp doctor", doctor.ExitWarnings, "", nil)
    }
    if len(result.Fixed) > 0 && len(result.Issues) > len(result.Fixed) {
        return camperrors.NewCommand("camp doctor", doctor.ExitPartialFix, "", nil)
    }
    return nil
}
```

Then find all call sites within `doctor.go` and change them to `return exitDoctorWithCode(result)`. In `runDoctor`, after sequence 09 task 01 has landed (which wires the JSON path through `exitDoctorWithCode`), the call sites should already be `return exitDoctorWithCode(result)` in both JSON and text modes. If task 09.01 has not landed yet, add the JSON-mode fix here as well:

```go
if doctorOpts.jsonOutput {
    if err := outputDoctorJSON(result); err != nil {
        return err
    }
    return exitDoctorWithCode(result)
}
outputDoctorText(result, doctorOpts.verbose, doctorOpts.fix)
return exitDoctorWithCode(result)
```

Remove the `"os"` import from `cmd/camp/doctor.go` if no longer used.

**Step 5: Replace `os.Exit` in `cmd/camp/clone.go`.**

Read the file to find all five sites. For each `os.Exit(clone.ExitXxx)`, replace with `return camperrors.NewCommand("camp clone", clone.ExitXxx, "", err)` where `err` is the local error variable at that point (or `nil` if there is no error in scope). Pay attention to the `determineCloneExitCode` function body near line 296: it currently returns `error` on success and calls `os.Exit` on failure. After the refactor it should return `error` for all paths.

**Step 6: Write the exit-code tests.**

Create `cmd/camp/exitcode_test.go`. Use the host `t.TempDir()` convention. The tests call a helper that invokes `Execute()` with the specific subcommand and arguments and captures the returned error, then checks `errors.As(err, &cmdErr)` and `cmdErr.ExitCode`. You will need to construct minimal fake campaign fixtures (init a temporary directory as a campaign root) to satisfy the commands' campaign detection.

Minimum test cases:

```go
func TestExitCodes(t *testing.T) {
    tests := []struct {
        name     string
        args     []string
        wantCode int
    }{
        // doctor with --json and a failing check should exit 1
        // sync with a broken sub should exit 1
        // clone partial should exit 3
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // set up fixture
            err := Execute() // with appropriate args
            var cmdErr *camperrors.CommandError
            if tt.wantCode == 0 {
                if err != nil {
                    t.Fatalf("unexpected error: %v", err)
                }
                return
            }
            if !errors.As(err, &cmdErr) || cmdErr.ExitCode != tt.wantCode {
                t.Fatalf("want exit %d, got err %v", tt.wantCode, err)
            }
        })
    }
}
```

The exact fixture setup (creating a `.campaign/` directory, a `campaign.yaml`) can be cribbed from existing tests in the package. Look for `testCampaignRoot` or similar helpers in `cmd/camp/*_test.go`.

### Edge cases

**Doctor ExitWarnings=1 equals ExitFailures=1:** Doctor's old distinction between warnings and failures is collapsed. If scripts or tests in the repo assert `== 2` for failures, those assertions must be updated. Run `grep -rn "doctor.ExitFailures\|ExitFailures = 2" .` to find them.

**Clone ExitPartialSuccess=3:** `docs/exit-codes.md` must clearly define exit 3 as "partial success" for clone so callers know some repos were cloned. This is the only command where 3 means partial success rather than partial repair.

**`camperrors.NewCommand` signature:** Verify the exact signature in `internal/errors/`. The review shows it as `camperrors.NewCommand(cmd string, exitCode int, stderr string, err error)`. The second positional arg is the exit code. Read the type before writing the calls.

### Out of scope

- UX-11 (doctor `--json` exits 0) is primarily sequence 09 task 01; this task only removes the `os.Exit` blocker.
- `camp version` using `Run` not `RunE` (UX-15) is sequence 12.02.
- Error message style (UX-16/17/18) is sequence 12.03.

## Done When

- [ ] All requirements met
- [ ] `docs/exit-codes.md` created and committed
- [ ] `grep 'os\.Exit' cmd/camp/sync.go cmd/camp/doctor.go cmd/camp/clone.go` returns no results
- [ ] `camp doctor --json` exits non-zero when checks fail (verify with `echo $?`)
- [ ] `camp clone` partial result exits 3 through `Execute()` return, confirmed by the new test
- [ ] Table-driven exit-code tests pass: `go test -count=1 -short -run TestExitCode ./cmd/camp` (both profiles)
- [ ] Both-profile gate passes