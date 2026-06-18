---
fest_autonomy: medium
fest_created: 2026-06-12T00:56:20.212199-06:00
fest_gate_id: testing
fest_gate_type: testing
fest_id: 06_testing.md
fest_managed: true
fest_name: Testing and Verification
fest_order: 6
fest_parent: 06_atomicity_and_locking
fest_status: completed
fest_tracking: true
fest_type: gate
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

- Project commit: `1ec923f` (`test: satisfy lint gate cleanup checks`).
- `just build stable` passed.
- `just build dev` passed.
- `just test unit` passed in the stable profile: 79 packages, 2560/2560 tests.
- `BUILD_TAGS=dev just test unit` passed: 81 packages, 2580/2580 tests.
- `just lint` passed with 0 issues after errcheck cleanup and no new host-fs test allowlist entries.
- `just vet` passed.
- `just test integration` passed on rerun: 368/368 tests in 168.25s.
- The first full integration run reported `TestPromote_InvalidStatus`; an isolated rerun of that test passed, and the full integration rerun passed cleanly.
