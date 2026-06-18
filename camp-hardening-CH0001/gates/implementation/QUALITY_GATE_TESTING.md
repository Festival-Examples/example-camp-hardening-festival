---
# Template metadata (for fest CLI discovery)
id: QUALITY_GATE_TESTING
aliases:
  - testing-verify
  - qg-test
description: Standard quality gate task for testing and verification

# Fest document metadata (becomes document frontmatter)
fest_type: gate
fest_id: <no value>
fest_name: Testing and Verification
fest_parent: <no value>
fest_order: <no value>
fest_gate_type: testing
fest_autonomy: medium
fest_status: pending
fest_tracking: true
fest_created: 2026-06-11T22:21:20-06:00
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

- [ ] `just test unit` passes in the stable profile
- [ ] Unit tests pass in the dev profile (`BUILD_TAGS=dev`)
- [ ] New/modified code has test coverage
- [ ] Tests are meaningful (not just coverage padding)

### Integration Tests

- [ ] `just test integration` passes for any touched container-tested path
- [ ] Components work together correctly

### Filesystem Safety

- [ ] Tests that create, copy, move, delete, chmod, git-init, or otherwise mutate real filesystem state run in a containerized integration harness
- [ ] Host-side unit tests do not use temp directories to exercise real filesystem workflows
- [ ] Unit tests remain limited to pure logic, mocks, or explicitly in-memory filesystem boundaries

### Error Handling

- [ ] Invalid inputs are rejected gracefully
- [ ] Error messages are clear and actionable
- [ ] Recovery paths work correctly

## Verification

- [ ] `just build stable` and `just build dev` both complete without warnings
- [ ] `just lint` and `just vet` are clean
- [ ] No regressions introduced (both profiles)
- [ ] Coverage meets project requirements
