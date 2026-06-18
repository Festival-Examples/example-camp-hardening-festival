# Baseline Gate Survey

**Date:** 2026-06-12
**Worktree commit:** 3654333
**Executor:** Grok agent (fest next loop)

## Results

| Command | Profile | Exit Code | Notes |
|---------|---------|-----------|-------|
| just build | stable | 0 | Listed build module recipes (modular justfile structure; `just build` enters build module default which lists). No binary produced by this literal invocation. |
| BUILD_TAGS=dev just build | dev | 0 | Listed build module recipes (same modular behavior). |
| just test unit | stable | 0 | All 2524 tests passed across 77 packages (31.82s). "✓ ALL 2524 TESTS PASSED". Manifest allowlist fix from task 01 effective. |
| go test -tags dev -count=1 -short ./... | dev | 0 | All packages passed (output truncated to last 50 lines in capture; every `ok` package, no FAIL). |
| just lint | stable | 0 | "0 issues." + "lint-no-host-fs-tests: clean (no NEW violators; 34 legacy files on allowlist)". |
| go vet -tags dev ./... | dev | 0 | Clean (no output). |
| go vet -tags=integration ./... | integration | 1 (expected) | `vet: cmd/camp/commit_integration_test.go:317:12: undefined: syncParentRef`. Belongs to 04_gate_enforcement (task 04_integration_tagged_test_migration per TEST-2 / R14). |

## Effective build invocations (for gate realization)

The standing matrix used literal `just build*` which under current modular justfile surface the module help rather than executing builds. The following were executed to fulfill the documented intent of verifying the build gates produce a successful binary in each profile:

- `just build stable` (stable): ✓ BUILD SUCCESSFUL (96 steps, 32.50s). Binary produced.
- `BUILD_TAGS=dev just build default-build` (dev): ✓ BUILD SUCCESSFUL (96 steps, 31.32s). Binary produced.

Both profiles build cleanly post task 01.

## Failures and owning sequences

- `go vet -tags=integration ./...`: FAIL (exit 1, expected at baseline). `cmd/camp/commit_integration_test.go:317:12: undefined: syncParentRef`. Owning sequence: 04_gate_enforcement (task 04_integration_tagged_test_migration per TEST-2 / R14 from the fable review).

No other failures in the executed matrix.

## Unexpected findings

None discovered during this survey. All standing gates (stable/dev unit, lint, dev vet, and effective builds) are green following the task 01 manifest allowlist bootstrap fix.

The only red item is the pre-known integration-tagged vet failure (TEST-2), already annotated to its owning sequence.

## Hard rule

No code changes were made in the worktree (or any other location outside this BASELINE.md and the prior task 01 edit which was already committed before this survey). All failures above are recorded for their owning sequences. Survey performed after `fest task completed` + `fest commit` of the prior task.

## Commands executed (verbatim from task)

1. `just build`
2. `BUILD_TAGS=dev just build`
3. `just test unit`
4. `go test -tags dev -count=1 -short ./...`
5. `just lint`
6. `go vet -tags dev ./...`
7. `go vet -tags=integration ./...`

(Plus the effective `just build stable` / `BUILD_TAGS=dev just build default-build` to satisfy "builds the camp binary" expectation in the task description.)
