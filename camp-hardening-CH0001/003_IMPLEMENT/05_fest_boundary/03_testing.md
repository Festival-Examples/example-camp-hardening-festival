---
fest_type: gate
fest_id: 03_testing.md
fest_name: Testing and Verification
fest_parent: 05_fest_boundary
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_gate_id: testing
fest_gate_type: testing
fest_managed: true
fest_created: 2026-06-12T00:56:20.210746-06:00
fest_updated: 2026-06-14T21:59:24.427146-06:00
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

### Results

Commands run from `projects/worktrees/camp/camp-hardening`:

```bash
just build stable
# PASS: 96 packages, build successful, total time 22.52s

just build dev
# PASS: 98 packages, build successful, total time 23.20s

just test unit
# PASS: 79 packages, 2544/2544 tests passed, total time 28.98s

BUILD_TAGS=dev just test unit
# PASS: 81 packages, 2564/2564 tests passed, total time 29.50s

just lint
# PASS: 0 issues; lint-no-host-fs-tests clean with 31 legacy files on allowlist

BUILD_TAGS=dev just lint
# PASS: 0 issues; lint-no-host-fs-tests clean with 31 legacy files on allowlist

just vet
# PASS: go vet ./...

just test integration
# PASS: 1 suite, 368/368 tests passed, total time 177.91s
```