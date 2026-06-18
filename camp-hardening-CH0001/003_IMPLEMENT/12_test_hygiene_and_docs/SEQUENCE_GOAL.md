---
fest_type: sequence
fest_id: 12_test_hygiene_and_docs
fest_name: test_hygiene_and_docs
fest_parent: 003_IMPLEMENT
fest_order: 12
fest_status: completed
fest_created: 2026-06-11T23:26:34.556113-06:00
fest_updated: 2026-06-15T16:49:34.213272-06:00
fest_tracking: true
---


# Sequence Goal: 12_test_hygiene_and_docs

**Sequence:** 12_test_hygiene_and_docs | **Phase:** 003_IMPLEMENT | **Status:** Pending | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Eliminate named flaky-test patterns, complete the help and docs polish batch, apply the R36 miscellaneous small fixes, and pin both external upstream dependencies to proper semver tags, leaving the suite green in both profiles and the CLI reference in sync with the code.

**Contribution to Phase Goal:** This is the closing sweep of 003_IMPLEMENT. All command-surface changes in sequences 09 and 10 precede this sequence, making the cli-reference regeneration here the final authoritative run. The flaky-test and hygiene fixes reduce noise in the gate so that 004_POLISH's verification pass works against a reliable signal. The dependency-tag task converts two pseudo-version pins to reproducible semver tags.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **Flaky test patterns resolved**: `internal/git/commit_test.go:940`, `internal/git/retry_test.go:30` use channel-based release; all five mtime-sleep sites converted to `os.Chtimes`; double Reset dropped from `tests/integration/main_test.go`; `internal/editor/editor_test.go` converted to `t.Setenv`; at least one `t.Chdir` replacement of a hand-rolled helper; `internal/git/commit/commit_test.go` host-git burn-down started with the ratchet allowlist shrinking by at least one entry; the named starter set of `tests/integration` Contains assertions converted to `--json` envelope assertions.
- [ ] **Help and docs polish applied**: root `promote` renamed to `shelve` (or folded into `dungeon move`) with direction-honest help; deprecated `wt` alias redirected; `camp status` real flags registered; shortcuts Use-string and wrong `camp projects` hint fixed; shell-init Long text updated; 22 Short-only workitem/workflow commands gain Long text; embedded Examples moved to the cobra `Example` field; `flow migrate` help contradiction fixed; `appendHistory` stub implemented (JSONL append) or removed with all advertisements; `just docs` regenerated with `BUILD_TAGS=''`; BR-9 first-release cap added to `tools/release-notes/main.go`.
- [ ] **Miscellaneous small fixes applied**: `camp commit --amend` no auto-stage; `--no-edit` flag added; index.lock minimum age floor; marker-before-symlink ordering; broken-symlink warning; dungeon crawl hint corrected; triage warnings moved to stderr; `pathsafe` merged into `pathutil`; shared slug package extracted; ID-scheme matrix doc written; UX-16/17/18/22 error-message batch applied.
- [ ] **Upstream dependency tags pinned**: `guild-scaffold` tagged at `v0.1.0` and `obey-shared` tagged at next semver on its `v0.4.x` line (minimum `v0.4.4`); `go.mod` updated to tags via `go get`; or a disposition recorded in the decision log if upstream access blocks tagging.

### Quality Standards

- [ ] **Both-profile gate passes**: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint` all green after each task.
- [ ] **CLI reference in sync**: `just docs` (with `BUILD_TAGS=''`) produces no diff after regeneration; `docs/cli-reference/camp_workitem_priority.md` exists.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_flaky_test_fixes | Eliminate named flaky patterns and begin host-git burn-down | Reliable green gate; ratchet allowlist shrinks |
| 02_help_docs_polish | Fix help text, move examples, regenerate CLI reference | Accurate help surface; no stale docs diff |
| 03_misc_small_fixes | Apply R36 batch plus UX error-message style fixes | Correct small behaviors; consistent error messages |
| 04_upstream_dependency_tags | Pin guild-scaffold and obey-shared to semver tags | Reproducible builds; no pseudo-version pins |

## Dependencies

### Prerequisites (from other sequences)

- Sequences 01-11: All command-surface changes must land before this sequence so the cli-reference regeneration in task 02 captures the complete final state. Specifically: sequence 09 and 10 changes make regeneration mandatory.
- Sequence 04 (gate enforcement): `just test unit`, `BUILD_TAGS=dev just test unit`, and `just lint` with `-tags=integration` must be wired before the verification plan here can use them as written.

### Provides (to other sequences)

- 004_POLISH verification pass: the final tree that 004_POLISH verifies comes from this sequence. The POLISH checklist includes `just gate` green end to end, `just docs` no-diff, and cli-reference currency.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| File:line drift from sequences 01-11 changes | Med | Low | Every task verifies targets before editing; re-locate by symbol if lines moved |
| `appendHistory` JSONL implementation is non-trivial | Low | Med | Scope is narrow: dungeon `AppendCrawlLog` is the exact pattern to copy; if blocked, removal path is fully documented in task 02 |
| guild-scaffold tagging blocked (no access or release process) | Med | Low | Task 04 has an explicit disposition path: record pseudo-version reasoning in the festival decision log |
| slug extraction causes import cycles | Low | Med | Extract to `internal/slug` (no upstream deps); both current packages are leaf packages with no shared importer chain |
| `just docs` diff after regeneration picks up noise from other sequences | Low | Low | Run `BUILD_TAGS=''` explicitly; sequence 02 task verifies no-diff immediately after regeneration |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: Flaky patterns eliminated, suite green in both profiles (task 01 complete)
- [ ] **Milestone 2**: Help/docs polish applied and `just docs` no-diff verified (task 02 complete)
- [ ] **Milestone 3**: R36 batch applied, upstream tags pinned, both-profile gate green (tasks 03 and 04 complete)

## Quality Gates

### Testing and Verification

- [ ] Both-profile gate green after each task: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`
- [ ] `just docs` (with `BUILD_TAGS=''`) produces no diff
- [ ] Named tests cited in each task pass

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [ ] Need another iteration? No
- [ ] If yes, new tasks created: N/A