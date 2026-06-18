---
fest_type: gate
fest_id: 05_testing.md
fest_name: Testing and Verification
fest_parent: 12_test_hygiene_and_docs
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_gate_id: testing
fest_gate_type: testing
fest_managed: true
fest_created: 2026-06-12T00:56:20.229211-06:00
fest_updated: 2026-06-15T16:42:02.542801-06:00
fest_tracking: true
fest_version: "1.0"
---


# Gate: Testing and Verification

Verify all functionality implemented in this sequence works correctly. All gates are LOCAL (just recipes in the camp worktree); never add or rely on GitHub Actions.

## Required Commands (run from the camp worktree root)

Run the full local gate in BOTH build profiles:

```bash
just build stable          # stable-profile build (BUILD_TAGS='')
just build dev             # dev-profile build (BUILD_TAGS=dev)
just test unit             # unit tests, stable profile
BUILD_TAGS=dev just test unit   # unit tests, dev profile (honored once 04.01/BR-1 lands)
just lint                  # golangci-lint new-from-merge-base + lint-no-host-fs-tests
just vet                   # go vet ./...
```

If the sequence touched container-tested paths (anything destructive or filesystem-mutating):

```bash
just test integration      # containerized integration tests (Docker/Colima)
```

## Test Categories

### Unit Tests

- [x] `just test unit` passes in the stable profile
- [x] Unit tests pass in the dev profile (`BUILD_TAGS=dev`)
- [x] New/modified code has test coverage
- [x] Tests are meaningful (not just coverage padding)

### Integration Tests

- [x] `just test integration` passes for any touched container-tested path
- [x] Components work together correctly

### Filesystem Safety

- [x] Tests that create, copy, move, delete, chmod, git-init, or otherwise mutate real filesystem state run in a containerized integration harness
- [x] Host-side unit tests do not use temp directories to exercise real filesystem workflows
- [x] Unit tests remain limited to pure logic, mocks, or explicitly in-memory filesystem boundaries

### Error Handling

- [x] Invalid inputs are rejected gracefully
- [x] Error messages are clear and actionable
- [x] Recovery paths work correctly

## Verification

- [x] `just build stable` and `just build dev` both complete without warnings
- [x] `just lint` and `just vet` are clean
- [x] No regressions introduced (both profiles)
- [x] Coverage meets project requirements

## Completion Notes

- Stable build passed: `GOFLAGS=-buildvcs=false just build stable`.
- Dev build passed: `GOFLAGS=-buildvcs=false BUILD_TAGS=dev just build dev`.
- Stable unit passed: `GOFLAGS=-buildvcs=false just test unit` with 81 packages and 2543/2543 tests passed.
- Dev unit passed on full rerun: `GOFLAGS=-buildvcs=false BUILD_TAGS=dev just test unit` with 83 packages and 2564/2564 tests passed. The first dev run hit one `internal/fsutil` failure; the isolated package passed immediately, and the full rerun passed.
- Stable and dev lint passed: `GOFLAGS=-buildvcs=false just lint` and `GOFLAGS=-buildvcs=false BUILD_TAGS=dev just lint`.
- Vet passed: `GOFLAGS=-buildvcs=false just vet`.
- Integration initially found that `TestAttachmentPin_RejectsExternalWithoutMarker` still expected the pre-UX-16 substring `outside campaign root`. Updated the assertion to the new clearer message text, `outside the campaign root`.
- Targeted integration passed after the assertion update: `GOFLAGS=-buildvcs=false go test -tags integration -run TestAttachmentPin_RejectsExternalWithoutMarker -v ./tests/integration`.
- Full integration passed after the assertion update: `GOFLAGS=-buildvcs=false just test integration` with 1 suite and 373/373 tests passed.