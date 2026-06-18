---
fest_type: sequence
fest_id: 09_app_contracts
fest_name: app_contracts
fest_parent: 003_IMPLEMENT
fest_order: 9
fest_status: completed
fest_created: 2026-06-11T23:26:34.50195-06:00
fest_updated: 2026-06-15T01:54:40.743601-06:00
fest_tracking: true
---


# Sequence Goal: 09_app_contracts

**Sequence:** 09_app_contracts | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Make every surface festival-app shells versioned, correct, and preflightable: exit codes honest, JSON payloads versioned, path semantics unified, profile field surfaced, and the manifest complete.

**Contribution to Phase Goal:** This sequence delivers the contract surface that festival-app updates against. Without it the app cannot parse intent listings, cannot tell a dev binary from stable, cannot trust doctor exit codes, and cannot know which commands are agent-safe. It is the prerequisite for the app's own schema-update work (tracked in the coordination workitem created in task 09.05).

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Doctor exit code**: `camp doctor --json` exits non-zero when any check fails or warns; JSON payload still emitted to stdout (task 01)
- [ ] **Flow items JSON**: `camp flow items --json` emits `workflow-items/v1alpha1` (or folded `workflow/v1`) versioned payload; errors to stderr, non-zero exit; no more dead `--json` flag (task 02)
- [ ] **Workitem create atomicity**: `camp workitem create` leaves no empty directory on partial failure; immediate retry after failure succeeds (task 03)
- [ ] **Intent JSON contract**: all four intent read commands plus `intent add` emit `intents/v1alpha1` through `jsoncontract`; multi-`--type` honored; `--format json` alias works; `-f` collision fixed (task 04)
- [ ] **Stage vocabulary**: typed `LifecycleStage` with `LifecycleStageNone`; `--stage none` finds design/explore items; `stage_vocabulary` in payload; `workitems/v1alpha6`; `omitempty` dropped from workflow ints/bools; festival-app coordination workitem created (task 05)
- [ ] **Path semantics**: intent/quest/status-all JSON uses campaign-relative paths plus `campaign_root`; EvalSymlinks applied at root; `docs/json-contracts.md` documents the join rule (task 06)
- [ ] **Manifest completeness**: every shipped command annotated; doctor annotation reflects read-vs-fix; manifest version bumped; completeness test added (task 07)
- [ ] **Version stamping and profile**: cross-platform recipes stamp ldflags; `ReadBuildInfo` fallback; `Profile` constant pair; `camp version --json` emits `schema_version` and `profile`; `--json` wins over `--short`; `RunE` (task 08)
- [ ] **Concepts JSON**: `camp concepts --json` emits `concepts/v1alpha1`; not-in-campaign exits non-zero (task 09)
- [ ] **docs/json-contracts.md**: updated rows for all new/bumped surfaces; scope statement added; join-rule documented

### Quality Standards

- [ ] **Both-profile gate**: `just build`, `just test unit`, and `just lint` pass in stable and dev profiles after every task
- [ ] **Schema version assertions**: tests assert the `schema_version` string for each new or bumped surface
- [ ] **Error envelope tests**: at least one test per surface confirms that a failure under `--json` emits the error envelope to stderr and exits non-zero

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Both-profile gate passes end to end
- [ ] `docs/json-contracts.md` updated rows verified
- [ ] Festival-app coordination workitem exists (created in task 05)

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_doctor_json_exit | Apply exit code after JSON output in doctor | Fixes the exit-0-on-failure bug that makes the app's health check meaningless |
| 02_flow_items_json | Implement versioned JSON for flow items | Gives the app a parseable flow item listing |
| 03_workitem_create_atomicity | Derive ref before mkdir; clean up on failure | Prevents the empty-dir leftover that blocks the app's FA0011-style retry |
| 04_intent_json_contract | Version all intent read surfaces; fix -f/-j collision | Gives the app a stable versioned intent contract; removes ambiguous shorthand |
| 05_stage_vocabulary | Typed stages, no-stage sentinel, v1alpha6 bump, coordination workitem | Fixes invisible design/explore items; notifies festival-app of all contract changes |
| 06_path_semantics_unification | campaign-relative paths + campaign_root everywhere | Gives the app a single join rule for all path fields |
| 07_manifest_annotations | Annotate all commands; bump manifest version | Gives the app a complete agent-allowed surface map |
| 08_version_stamping_and_profile | ldflags, ReadBuildInfo, Profile constant, version --json | Lets the app preflight dev-only commands and identify binary channel |
| 09_concepts_json | Version concepts --json; fix not-in-campaign exit | Gives the app a safe concepts surface |

## Dependencies

### Prerequisites (from other sequences)

- **01_gate_restoration**: Unit gate must be green before meaningful test output from this sequence. Sequence 01 adds `workitem priority` to the manifest allowlist; coordinate task 07's count updates with that change already in place.
- **04_read_safety** (sequence 04 in 003_IMPLEMENT): intent read commands stop calling `EnsureDirectories`; this sequence builds the new JSON contract on top of reads that no longer mutate state.
- **06_locks** (sequence 06 in 003_IMPLEMENT): workitem store mutations are locked before this sequence adds new JSON surfaces that may trigger concurrent reads.

### Provides (to other sequences)

- **The `Profile` constant** (tasks 08): required by sequence 10.05 (channel-aware scaffolded templates and README flow caveat per D002).
- **The versioned contract surface** (all tasks): the festival-app coordination workitem (task 05) is the handoff artifact that drives the app's own update work.
- **The `docs/json-contracts.md` stability matrix** (tasks 04-09): feeds the 004_POLISH phase's JSON-6 verification checklist.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root). Branch point `bc2ad1f`. All file:line references in task documents were verified against this worktree; note that the second-pass review at `11c270f` added intent/TUI work that may cause line drift on intent-related files. Verify each target line before editing.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Line drift from intent/TUI commits between `bc2ad1f` and working HEAD | Medium | Low | Read each file before editing; the task documents cite verified lines; update references as needed |
| `WorkItem.LifecycleStage` type change (string -> LifecycleStage) causes compile failures in callers | Medium | Medium | Run `grep -r "LifecycleStage" ./internal` before building after task 05; fix each site |
| `WorkItemWorkflow` omitempty removal changes the JSON shape and breaks app-side assumptions | Low | Medium | The app does not currently parse WorkItemWorkflow (v1alpha5 is the current contract); the bump to v1alpha6 is the notification mechanism |
| Manifest count assertions in test fail due to interaction with sequence 01's already-applied fix | Medium | Low | Read `manifest_test.go` counts before updating; add rather than replace assertions |
| `debug.ReadBuildInfo()` returns empty on some build paths | Low | Low | The fallback only runs when `Version == "dev"`; if ReadBuildInfo returns empty, `dev` is preserved as before |

## Progress Tracking

### Milestones

- [ ] **Milestone 1 (exit-code and atomicity baseline)**: Tasks 01-03 done; doctor exit, flow items JSON, and create atomicity correct; both-profile gate green
- [ ] **Milestone 2 (intent and stage contracts)**: Tasks 04-06 done; intent versioned, stages fixed, paths unified; `docs/json-contracts.md` rows added for these surfaces
- [ ] **Milestone 3 (manifest, version, concepts, coordination)**: Tasks 07-09 done; manifest complete, profile field surfaced, concepts safe; festival-app coordination workitem created

## Quality Gates

### Testing and Verification

- [ ] Both-profile gate (`just build` + `just test unit` + `just lint` in stable and dev) passes after each task
- [ ] `go vet -tags=integration ./...` clean
- [ ] Schema version string asserted in tests for each new or bumped surface

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified (context propagation, camperrors not fmt.Errorf, snake_case JSON)

### Iteration Decision

- [ ] Need another iteration? No (all scope is bounded; out-of-scope items called out per task)
- [ ] If yes, new tasks created: N/A

## Out of Scope

- Standardizing `--json` flag across non-app commands (`--format` commands not on the festival-app surface): sequence 10.01.
- Renaming `--format` flags beyond the intent deprecated-alias: sequence 10.01.
- Replacing `os.Exit` in doctor's text path: sequence 10.02.
- `camp status all` full versioning (status-all/v1alpha1 envelope): sequence 10.01.
- workitem/workflow JSON determinism and nil-guard fixes: verified clean in second-pass-fable.md "hunted and found clean" list; no action needed here.