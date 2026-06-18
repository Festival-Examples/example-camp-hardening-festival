---
fest_type: task
fest_id: 02_baseline_gate_survey.md
fest_name: baseline_gate_survey
fest_parent: 01_gate_restoration
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.602556-06:00
fest_updated: 2026-06-12T13:24:09.393187-06:00
fest_tracking: true
---


# Task: Baseline Gate Survey

## Objective

Run the full standing verification matrix in both build profiles from the worktree root, record every command's pass/fail result in a `BASELINE.md` file in this sequence directory, and note the owning sequence for each failure. No fixes are made in this task.

## Requirements

- [ ] Every command in the standing verification matrix is executed from `projects/worktrees/camp/camp-hardening` (the worktree root). Commands are run in both stable and dev profiles where applicable.
- [ ] A `BASELINE.md` file is created in the sequence directory (`festivals/planning/camp-hardening-CH0001/003_IMPLEMENT/01_gate_restoration/BASELINE.md`) containing the exact command, its exit code, and a summary of output or failure for every entry.
- [ ] Each failing command has an "owning sequence" annotation (e.g., "belongs to 04_gate_enforcement") so later sequences can track which baseline failures they are responsible for resolving.
- [ ] The hard rule: no code changes, no `go.mod` changes, no test fixes are made in this task. If a failure is discovered that is NOT already in the known findings, that gap is noted in BASELINE.md and an intent is created for it after the survey. The executor must not edit any file outside `festivals/planning/camp-hardening-CH0001/003_IMPLEMENT/01_gate_restoration/BASELINE.md`.

## Implementation

### Background

The worktree at `projects/worktrees/camp/camp-hardening` is at commit `bc2ad1f` (PR #318 merge). Task 01 of this sequence fixed the manifest allowlist, making the unit gate green. This task takes a complete snapshot of every quality gate before any other code change lands in the festival. The snapshot gives sequences 02-12 a clean before/after comparison.

The standing verification commands are defined in `IMPLEMENTATION_PLAN.md` (section "Standing verification commands"). They are reproduced here in the order the executor must run them.

### Commands to run

Execute each command from the worktree root (`projects/worktrees/camp/camp-hardening`). Record the command, exit code, and a brief description of any failures.

**Note:** task 01 must be completed (merged, committed) before running this survey, because `just test unit` calls `go test ./cmd/camp` which includes the manifest test. If task 01 is not yet committed, the manifest test will still be red.

**1. Stable build**

```bash
just build
```

Expected: builds the `camp` binary in the stable profile (no build tags). Record pass or fail.

**2. Dev build**

```bash
BUILD_TAGS=dev just build
```

Expected: builds the `camp` binary with `//go:build dev` files included. Record pass or fail.

**3. Stable unit tests**

```bash
just test unit
```

This invokes `go run ./internal/buildutil test` which runs the stable-profile unit suite with `-count=1 -short` across all packages (parallel per package). Record pass or fail and capture any FAIL lines.

**4. Dev unit tests**

The `just test unit` recipe does not yet honor `BUILD_TAGS` (that gap is sequence 04's target). Until sequence 04 lands, run dev-profile tests directly:

```bash
go test -tags dev -count=1 -short ./...
```

Record pass or fail. If specific packages fail, list them.

**5. Stable lint**

```bash
just lint
```

This runs golangci-lint against new branch issues from `origin/main` plus `just lint-no-host-fs-tests`. Record pass or fail and capture any lint findings.

**6. Dev vet**

```bash
go vet -tags dev ./...
```

Expected to pass unless a dev-tagged file has a vet error. Record pass or fail.

**7. Integration-tagged vet**

```bash
go vet -tags=integration ./...
```

This command is expected to FAIL at baseline. The known cause is `cmd/camp/commit_integration_test.go:317: undefined: syncParentRef` (TEST-2 in the findings; the function was removed when submodule sync was pulled from `camp p commit`). Record the failure verbatim. Owning sequence: 04_gate_enforcement (task 04_integration_tagged_test_migration).

### BASELINE.md format

Create `BASELINE.md` in the sequence directory with this structure:

```
# Baseline Gate Survey

**Date:** <date of execution>
**Worktree commit:** bc2ad1f
**Executor:** <name or agent id>

## Results

| Command | Profile | Exit Code | Notes |
|---------|---------|-----------|-------|
| just build | stable | 0 | pass |
| BUILD_TAGS=dev just build | dev | 0 | pass |
| just test unit | stable | <code> | <summary> |
| go test -tags dev -count=1 -short ./... | dev | <code> | <summary> |
| just lint | stable | <code> | <summary> |
| go vet -tags dev ./... | dev | <code> | <summary> |
| go vet -tags=integration ./... | integration | 1 (expected) | cmd/camp/commit_integration_test.go:317: undefined: syncParentRef. Belongs to 04_gate_enforcement |

## Failures and owning sequences

List each failing command here with the owning sequence that will resolve it:

- `go vet -tags=integration ./...`: FAIL. `cmd/camp/commit_integration_test.go:317: undefined: syncParentRef`. Owning sequence: 04_gate_enforcement (task 04_integration_tagged_test_migration per TEST-2 / R14)
- `<any additional failures found>`: `<description>`. Owning sequence: `<sequence id>`

## Unexpected findings

Any failures not already covered by R1-R36 or N-1-N-27 that are discovered during this survey must be listed here with enough detail (file, line, error text) for triage. Do not fix them. Create a `camp workitem create --type design` item after the survey completes.

## Hard rule

No code changes were made during this task. All failures above are recorded for their owning sequences.
```

Fill in the actual exit codes and output. If a command produces a long failure, include the first 10-20 lines of the error and a count of total failures.

### Known expected state after task 01

After task 01 is committed:

- `just test unit` (stable): pass. The manifest test failure (TEST-1) is resolved by task 01.
- `go test -tags dev -count=1 -short ./...`: pass for the manifest tests. Other packages: unknown until the survey runs.
- `go vet -tags=integration ./...`: FAIL at `cmd/camp/commit_integration_test.go:317` (TEST-2; expected baseline failure; belongs to sequence 04).
- All other commands: unknown until the survey runs; their actual state is exactly what this task documents.

## Done When

- [ ] All requirements met
- [ ] `BASELINE.md` exists at `festivals/planning/camp-hardening-CH0001/003_IMPLEMENT/01_gate_restoration/BASELINE.md`
- [ ] Every command in the standing verification matrix appears in the BASELINE.md results table with an actual exit code
- [ ] Every failing command has an owning-sequence annotation
- [ ] No files were modified in the worktree or any other location outside `BASELINE.md`