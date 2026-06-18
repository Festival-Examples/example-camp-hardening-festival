---
fest_type: task
fest_id: 01_doctor_json_exit.md
fest_name: doctor_json_exit
fest_parent: 09_app_contracts
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.750335-06:00
fest_updated: 2026-06-15T00:33:06.101718-06:00
fest_tracking: true
---


# Task: Fix `camp doctor --json` exit code on failures (R8 / UX-11)

## Objective

Apply the correct non-zero exit code path after JSON output so that `camp doctor --json` exits non-zero when checks fail, matching the documented exit-code table.

## Requirements

- [ ] `camp doctor --json` exits non-zero when any check reports a failure or warning, using the same code table already documented in the Long help (1 = warnings, 2 = failures, 3 = partial fix)
- [ ] The JSON payload is still emitted to stdout before the non-zero exit; the exit code is the only change
- [ ] The fix uses a typed `camperrors.CommandError` (preferred) rather than a direct `os.Exit` call inside RunE, consistent with UX-14's direction to eliminate `os.Exit`; note that fully removing all `os.Exit` from doctor is sequence 10.02 scope, this task only makes `--json` consistent
- [ ] A test covers the failing-check-under-`--json` path and asserts both non-zero exit and valid JSON on stdout

## Implementation

### Background

`cmd/camp/doctor.go` (verified at worktree bc2ad1f) at lines 114-121:

```go
if doctorOpts.jsonOutput {
    return outputDoctorJSON(result)
}
outputDoctorText(result, doctorOpts.verbose, doctorOpts.fix)
return exitDoctorWithCode(result)
```

`outputDoctorJSON` (`:137-141`) encodes the result and returns nil (or a JSON encoding error). `exitDoctorWithCode` (`:221-236`) calls `os.Exit` based on `result.Failed`, `result.Warned`, and fixed/remaining counts. The JSON branch returns before `exitDoctorWithCode` is ever reached, so `--json` always exits 0 on check failures.

The `doctor` package exposes `ExitFailures`, `ExitWarnings`, `ExitPartialFix` constants (used in `exitDoctorWithCode`). The `camperrors` package provides `CommandError` with an `ExitCode` field; `cmd/camp/main.go` maps that to the process exit code. This is the standard path for all other commands.

### Step 1: update `runDoctor` to apply exit code after JSON output

In `cmd/camp/doctor.go`, change the JSON branch from:

```go
if doctorOpts.jsonOutput {
    return outputDoctorJSON(result)
}
```

to:

```go
if doctorOpts.jsonOutput {
    if err := outputDoctorJSON(result); err != nil {
        return err
    }
    return exitDoctorWithCodeE(result)
}
```

The new helper `exitDoctorWithCodeE` returns a `*camperrors.CommandError` instead of calling `os.Exit`, so cobra's error path handles the exit code:

```go
func exitDoctorWithCodeE(result *doctor.DoctorResult) error {
    code := 0
    if result.Failed > 0 {
        code = doctor.ExitFailures
    } else if result.Warned > 0 {
        code = doctor.ExitWarnings
    } else if len(result.Fixed) > 0 && len(result.Issues) > len(result.Fixed) {
        code = doctor.ExitPartialFix
    }
    if code == 0 {
        return nil
    }
    return camperrors.NewCommand("doctor", code, "", nil)
}
```

Keep the existing `exitDoctorWithCode` (which calls `os.Exit`) unchanged for the text path; that is the scope of sequence 10.02.

The `doctor.go` annotation at lines 52-55 sets `agent_allowed: false` with reason "Has --fix mode that is destructive". The read path under `--json` is separate concern. Sequence 09.07 (manifest annotations) handles the annotation split for doctor read vs `--fix`; do not change annotations here.

### Step 2: add test

In `cmd/camp/` (alongside existing doctor tests or a new `doctor_test.go`), add a test that:

1. Creates a minimal campaign fixture with a broken submodule (or a fixture that causes any doctor check to fail).
2. Invokes `doctorCmd.Execute()` with `--json` and captures stdout and the returned error.
3. Asserts the returned error is non-nil with exit code matching the failure type.
4. Asserts stdout parses as valid JSON (unmarshal into `map[string]any` is sufficient; do not hard-code the shape).

The simplest fixture for a failing check is an orphaned gitlink entry in the git index. Alternatively, mock the `doctor.Doctor` result if the package exposes an interface. Check how the test file for the manifest command constructs its fixture for the level of setup needed.

## Done When

- [x] All requirements met
- [x] `cmd/camp/doctor.go` JSON branch calls `exitDoctorWithCodeE` after `outputDoctorJSON`
- [x] `exitDoctorWithCodeE` added and returns `camperrors.CommandError` with the correct code
- [x] Both-profile gate passes (`just build` and `just test unit` in stable and dev profiles)
- [x] Test exists that asserts non-zero exit plus valid JSON stdout under `--json` with failing checks
