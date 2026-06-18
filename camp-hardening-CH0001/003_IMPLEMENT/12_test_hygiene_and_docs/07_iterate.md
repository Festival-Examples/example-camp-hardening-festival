---
fest_type: gate
fest_id: 07_iterate.md
fest_name: Review Results and Iterate
fest_parent: 12_test_hygiene_and_docs
fest_order: 7
fest_status: completed
fest_autonomy: medium
fest_gate_id: iterate
fest_gate_type: iterate
fest_managed: true
fest_created: 2026-06-12T00:56:20.2304-06:00
fest_updated: 2026-06-15T16:49:07.99401-06:00
fest_tracking: true
fest_version: "1.0"
---


# Gate: Review Results and Iterate

Address all findings from testing and code review. Iterate until the sequence meets quality standards.

## Findings to Address

### From Testing

- [x] `TestAttachmentPin_RejectsExternalWithoutMarker` expected the previous UX-16 wording (`outside campaign root`). Updated the assertion to match the new clearer error text (`outside the campaign root`) and reran the targeted plus full integration suites.

### From Code Review

- [x] `project.AddLinked` rollback removed a pre-existing same-campaign `.camp` marker if symlink creation failed. Fixed rollback to restore the prior marker bytes and added `TestAddLinked_RestoresExistingMarkerWhenSymlinkFails`.

## Iteration

For each finding:

1. Fix the issue
2. Re-run affected tests
3. Verify linting passes

## Definition of Done

- [x] All critical findings fixed
- [x] All tests pass after changes
- [x] Linting passes
- [x] Code review findings addressed
- [x] Ready to commit

## Verification

- Testing fix: `GOFLAGS=-buildvcs=false go test -tags integration -run TestAttachmentPin_RejectsExternalWithoutMarker -v ./tests/integration`; `GOFLAGS=-buildvcs=false just test integration`
- Review fix: `GOFLAGS=-buildvcs=false go test ./internal/project`; `GOFLAGS=-buildvcs=false just test unit`; `GOFLAGS=-buildvcs=false just lint`; `git diff --check`
- Latest pushed head: `5db31ec9753d1c44a3e6dd69229223405a16cfc4`