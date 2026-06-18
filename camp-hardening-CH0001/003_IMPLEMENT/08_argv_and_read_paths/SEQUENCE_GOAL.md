---
fest_type: sequence
fest_id: 08_argv_and_read_paths
fest_name: argv_and_read_paths
fest_parent: 003_IMPLEMENT
fest_order: 8
fest_status: completed
fest_created: 2026-06-11T23:26:34.483952-06:00
fest_updated: 2026-06-15T00:28:27.311474-06:00
fest_tracking: true
---


# Sequence Goal: 08_argv_and_read_paths

**Sequence:** 08_argv_and_read_paths | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Eliminate all argv-flattening bugs in the camp run and flow runner dispatch paths so arguments survive byte-exact, and eliminate all filesystem writes from read-only commands so festival-app polling, `cr` daily workflows, and concurrent intent reads are safe.

**Contribution to Phase Goal:** The `cr` shell alias is the primary daily driver for all project dispatch; corrupted arguments under that path produce silent wrong-behavior in every festival execution loop. The workitem list prune side effect and the intent EnsureDirectories migration are the two mechanisms by which reads from festival-app polling permanently alter campaign state. Fixing these four bugs closes the "read mutates state" class that the second pass (N-8, N-18) and the first pass (R5, R18) both flagged, and puts festival-app polling on a stable foundation.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Byte-exact argv dispatch**: `cr camp commit -m "fix: two words"` delivers the two-word message to just without splitting; args containing `$VAR`, `;`, `*`, single quotes, double quotes, and leading dashes all survive the project just-dispatch path.
- [ ] **Safe flow runner arg splicing**: `camp flow run <entry> -- "us east"` delivers one positional to the registered shell command; `$(...)` in an extra arg is data, not code.
- [ ] **Read-only workitem list**: `camp workitem list` (human and `--json`) leaves `.campaign/settings/workitems.json` byte-for-byte unchanged on disk regardless of transient discovery gaps; `git status` reports no change after a read.
- [ ] **Read-only intent commands**: `camp intent list`, `find`, `count`, and `show` against a legacy-layout fixture produce correct output or a clear warning, but mutate nothing on disk; `git status` reports no change after any of them.
- [ ] **Honest status cache**: `camp status all --json` writes no cache file; the `--no-cache` flag is either removed along with the write-on-read, or a real cache-read path is added that honors it.

### Quality Standards

- [ ] **Both-profile gate**: all five deliverables verified under both stable (`just build && just test unit && just lint`) and dev (`BUILD_TAGS=dev just build && go test -tags dev ./... && BUILD_TAGS=dev just lint`) profiles.
- [ ] **Zero git-status drift**: a dedicated read-only assertion test runs list/find/count/show/status-all against a fixture and asserts `git status --porcelain` produces empty output.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_just_dispatch_argv | Exec `just` directly with argv for project dispatch; keep `sh -c` only for raw-command form | Eliminates the `strings.Join` flattening that breaks `cr` daily workflows |
| 02_flow_runner_positionals | Pass extra args as shell positionals using `sh -c '<cmd> "$@"' sh args...` | Eliminates shell injection through extra args while keeping the registry command's shell semantics |
| 03_workitem_list_read_only | Remove `priority.Prune` from the list RunE; move it to explicit write paths | Stops transient discovery gaps permanently deleting priorities during reads festival-app polls |
| 04_intent_read_migration_and_status_cache | Stop calling `EnsureDirectories` from read commands; stop writing the status cache unconditionally | Eliminates the two remaining read-mutates-state sites: intent migration and status-all cache write |

## Dependencies

### Prerequisites (from other sequences)

- Sequence 01 (gate_restoration): unit gate must be green in both profiles before this sequence's tests are meaningful.
- Sequence 04 (gate_enforcement): `priority.WithLock` is introduced in sequence 06 task 04; task 03 of this sequence depends on `WithLock` existing as the sanctioned mutation door, and the pruning moved here must also use it.

### Provides (to other sequences)

- Byte-exact argv dispatch: relied on by the `cr` daily workflow used throughout festival execution and by any sequence that validates behavior via `camp run`.
- Read-path purity: festival-app polling correctness depends on `camp workitem --json` and `camp intent list -f json` leaving no disk side effects; this guarantee is required before the sequence 09 contract tests can be trusted.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| File:line drift from worktree divergence since review baseline `bc2ad1f` | Low | Low | Every task requires verifying file:line against the worktree before editing; all lines confirmed current in this document |
| `ExecuteCommand` callers relying on the sh -c join for raw-command semantics break | Medium | Medium | Fix only the project just-dispatch branch; preserve `sh -c` for raw-command form and all other callers |
| `priority.WithLock` not yet merged when task 03 executes | Low | Low | Task 03 notes the dependency; if sequence 06 task 04 has not landed, use a direct `fsutil.AcquireFileLock` call and document it as a cleanup target |
| Legacy-layout campaigns in active use encounter the warning instead of silent migration | Low | Low | The warning points to `camp init --repair`; that path is well-tested and always was the correct migration trigger |
| Removing `writeStatusCache` breaks a consumer that reads the cache | Very Low | Medium | Second-pass hunt confirmed no cache-read path exists anywhere in camp; the decision to delete is correct but documented |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: Tasks 01 and 02 complete; `cr` dispatch and flow runner passing metacharacter tests in both profiles.
- [ ] **Milestone 2**: Task 03 complete; `camp workitem --json` confirmed write-free against a fixture with a transient discovery gap.
- [ ] **Milestone 3**: Task 04 complete; all four intent read commands and `camp status all` write nothing; both-profile gate green.

## Quality Gates

### Testing and Verification

- [ ] All unit tests pass (stable and dev profiles)
- [ ] Integration tests complete (read-only assertion fixture test passes)
- [ ] Named acceptance test: `cr just <recipe> -m "two words"` delivers the two-word string

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [ ] Need another iteration? No
- [ ] If yes, new tasks created: N/A