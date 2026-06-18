---
fest_type: sequence
fest_id: 05_fest_boundary
fest_name: fest_boundary
fest_parent: 003_IMPLEMENT
fest_order: 5
fest_status: completed
fest_created: 2026-06-11T23:26:34.432391-06:00
fest_updated: 2026-06-14T22:14:01.608369-06:00
fest_tracking: true
---


# Sequence Goal: 05_fest_boundary

**Sequence:** 05_fest_boundary | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Prevent camp from ever relocating, adopting, or writing into fest-owned state by adding fest-ownership guards to the dungeon triage and resolver layers and removing the promote command's MkdirAll of fest-internal festival ingest directories.

**Contribution to Phase Goal:** The implementation phase must harden every camp subsystem that touches the campaign filesystem. This sequence eliminates the two confirmed boundary violations (WF-1, WF-2) where camp dungeon can rename or adopt fest's state tree, and the third (WF-7) where intent promote reaches behind fest to write fest-internal directory structure directly. These guards protect every later sequence's dungeon and promote interactions from silently corrupting fest state.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Dungeon exclusion map hardened**: `festivals` and `projects` added to the built-in exclusion map in `internal/dungeon/service_list.go:125-136`; a default `dungeon/.crawl.yaml` is scaffolded at campaign init via `internal/dungeon/scaffold/init.go` as defense in depth; regression tests confirm neither directory appears as a triage candidate from a campaign-root fixture.
- [ ] **Resolver fest-ownership check**: `internal/dungeon/resolver.go:64-86` walking-up logic skips any `dungeon/` directory whose parent chain contains a `festivals/` path element; a regression test with cwd inside `festivals/planning/<fest>/` confirms the resolver returns the campaign dungeon (or none), never `festivals/dungeon`.
- [ ] **Promote ingest boundary enforced**: `copyIntentToIngest` in `internal/intent/promote/promote.go:344-359` no longer calls `os.MkdirAll` on `001_INGEST/input_specs`; when the directory is absent the copy is skipped with a clear user notice; fest CLI stderr is surfaced when `createFestival` fails; a fest workitem capturing the missing `fest ingest <file>` capability is created via `camp workitem create`.

### Quality Standards

- [ ] **Both-profile gate passes**: `just build`, `BUILD_TAGS=dev just build`, `just test unit` (or `go test -count=1 -short ./...`), `go test -tags dev -count=1 -short ./...`, `just lint`, and `BUILD_TAGS=dev just lint` all green.
- [ ] **Named regression tests present and passing**: at minimum `TestListParentItems_ExcludesFestivals`, `TestListParentItems_ExcludesProjects`, `TestResolveContext_IgnoresFestivalsDungeon`, `TestCopyIntentToIngest_SkipsAbsentIngestDir`, and `TestCreateFestival_SurfacesStderr`.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_dungeon_exclusion_and_resolver_guards | Add `festivals` and `projects` to the built-in dungeon exclusion map, make the resolver skip fest-owned dungeon directories, and scaffold a default `.crawl.yaml` at campaign init | Closes WF-1 and WF-2: prevents triage from offering fest's state tree and prevents the resolver from adopting `festivals/dungeon/` as a camp dungeon context |
| 02_promote_ingest_boundary | Stop `copyIntentToIngest` from MkdirAll-ing fest-internal directories, surface fest CLI stderr on failure, and create the fest workitem for a `fest ingest` command | Closes WF-7: camp no longer writes into fest-owned directory structure; user gets actionable guidance when ingest has not been run; the missing capability is tracked in the right place |

## Dependencies

### Prerequisites (from other sequences)

- Sequence 01 (01_gate_restoration): unit gate must be green in both profiles before this sequence runs; the regression tests added here must pass under a working gate.
- Sequence 04 (04_gate_enforcement): trustworthy `just gate` in both profiles; until sequence 04 lands, use the raw `go test -tags dev` form documented in IMPLEMENTATION_PLAN.md.

### Provides (to other sequences)

- Fest-ownership guards protecting the dungeon exclusion map and resolver: every later sequence that exercises dungeon triage, dungeon move, or cwd-based dungeon resolution (sequences 06, 08, 11) operates on a tree where `festivals/` and `projects/` are structurally excluded and can never be accidentally relocated.
- The fest workitem created in task 02 establishes the canonical missing-capability record for `fest ingest <file>`, so future work (sequence 11 or a dedicated fest festival) can depend on it rather than re-deriving the requirement.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Resolver path element check fails on macOS symlinked `/var` paths | Med | High | Use `filepath.EvalSymlinks` before splitting; the worktree's existing resolver already calls EvalSymlinks at lines 55-62 before the walk loop, so canonical paths are guaranteed inside the loop |
| Adding `festivals` to the exclusion map silently breaks a campaign that legitimately has a `festivals` directory that is NOT fest-owned | Low | Low | The exclusion only applies to triage candidates from the campaign root; any directory named `festivals` at the campaign root is by convention fest's; the fix is correct for any real campaign |
| Scaffolding `.crawl.yaml` with an `excludes` list at init overwrites a user-customized file | Low | Med | Use the same skip-if-exists idempotency pattern as OBEY.md and .gitkeep in `internal/dungeon/scaffold/init.go`; never overwrite when `Force` is false |
| Removing MkdirAll from `copyIntentToIngest` silently skips the copy when `001_INGEST/input_specs` does not yet exist, surprising users who expect it to work | Med | Low | Print a clear notice to stderr naming the missing directory and instructing the user to run `fest` ingest; the skip-with-notice behavior is more correct than silently creating orphan files |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: Exclusion map and crawler guard landed with passing regression tests (task 01 done).
- [ ] **Milestone 2**: Resolver fest-ownership check landed with cwd-in-festivals regression test passing (task 01 done).
- [ ] **Milestone 3**: Promote ingest boundary enforced, fest stderr surfaced, fest workitem created (task 02 done); both-profile gate green end to end.

## Quality Gates

### Testing and Verification

- [ ] `go test -count=1 -short ./internal/dungeon/...` passes in both profiles
- [ ] `go test -count=1 -short ./internal/intent/promote/...` passes in both profiles
- [ ] Named regression tests all present and green

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [ ] Need another iteration? No