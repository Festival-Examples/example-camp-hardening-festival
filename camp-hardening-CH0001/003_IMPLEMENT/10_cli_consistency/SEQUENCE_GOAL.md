---
fest_type: sequence
fest_id: 10_cli_consistency
fest_name: cli_consistency
fest_parent: 003_IMPLEMENT
fest_order: 10
fest_status: completed
fest_created: 2026-06-11T23:26:34.519833-06:00
fest_updated: 2026-06-15T03:25:28.756969-06:00
fest_tracking: true
---


# Sequence Goal: 10_cli_consistency

**Sequence:** 10_cli_consistency | **Phase:** 003_IMPLEMENT | **Status:** Completed | **Created:** 2026-06-11T23:26:34-06:00

## Sequence Objective

**Primary Goal:** Produce a consistent, agent-trustworthy CLI surface: one JSON flag idiom, one exit-code table, signal-aware root context, honest launch docs, transparent shell wrappers, and guarded TTY commands, so every consumer from a human shell to festival-app gets predictable behavior.

**Contribution to Phase Goal:** Sequences 01-09 fix data-loss, gate, boundary, and contract issues. This sequence closes the consistency gap that makes the otherwise-fixed surface unpredictable for callers: flag spellings differ, exit codes contradict each other across commands, Ctrl-C cannot cancel long operations, scaffolded skills instruct dev-only commands, shell wrappers discard errors, and four TUI commands panic on non-TTY. Fixing all of these in one sequence prevents agents from having to know which subtree they are calling.

## Success Criteria

The sequence goal is achieved when:

### Required Deliverables

- [ ] **JSON flag unification**: `--json` aliased on every `--format` command; `flow add --json` renamed `--from-json`; all remaining `--json` commands wrapped in `jsoncontract.RunE`; `status all --json` empty case emits `{"schema_version":"status-all/v1alpha1","timestamp":...,"repos":[]}` not a human string.
- [ ] **Exit-code table**: one documented CLI-wide table (0 ok, 1 runtime failure, 2 usage error, 3+ partial states); `internal/doctor/options.go`, `internal/sync/options.go`, `internal/clone/options.go` aligned; every `os.Exit` in `cmd/camp/sync.go`, `doctor.go`, and `clone.go` replaced with `camperrors.CommandError`; table in `docs/`.
- [ ] **Signal context**: `signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)` in `main.go`, `rootCmd.ExecuteContext(ctx)` in `root.go`; nil-ctx fallbacks deleted; `internal/git/retry.go` backoff sleep replaced with a `select` that returns on `ctx.Done()`; reachable ERR-4 helper sites threaded with the root context.
- [ ] **Alias and usage fixes**: `reg` alias removed from `register.go`; `SilenceUsage` set globally on root with per-command patches removed; `SetFlagErrorFunc` reinstates usage for parse errors with exit 2; dead `--config` deleted or wired; dead `--verbose` global deleted and local shadows removed.
- [ ] **README and flow channel**: `README.md` install path corrected to `.../camp/cmd/camp@latest`; scaffolded skills and templates render flow instructions only under dev profile using the `Profile` constant from sequence 09 task 08; README documents `flow` as dev-channel; `dungeon add` Long help drops the `camp flow init` reference; `docs` justfile recipe pins `BUILD_TAGS=''`.
- [ ] **Shell and nav fixes**: go/switch shell wrappers drop `2>/dev/null`, check exit codes, show the generic message only on truly empty output; `--help`/`--json`/`--print` passthrough added to go and switch branches; `go.go` stats the resolved path before printing, forces one index rebuild on miss, rejects `--list` with `--print`, drops the `!printOnly` disambiguation gate; nav cache rebuild locked with `fsutil.AcquireFileLock`; save errors surface as warnings.
- [ ] **Repair failures and TTY guards**: registry-save failure during init returns the error; `fest init` failure distinguishes not-installed vs failed and prints the cause; `repair.go` and `project/add.go` swallowed cleanup calls warn on stderr; `intent explore`, `settings`, `intent crawl`, and `dungeon crawl` each gain an `IsTerminal` guard and `agent_allowed: false` annotation.
- [ ] **Pull/push flag handling**: `ExtractSubFlags` intercepts only `--project`/`--project=`, not bare `-p`; `--` terminator honored; `pull.go` error hint corrected to `projects/<path>` form.

### Quality Standards

- [ ] **Both-profile gate passes**: `just build`, `just test unit` (or `BUILD_TAGS=dev just test unit`), and `just lint` all pass in both stable and dev profiles after every task.
- [ ] **Named tests present**: each task's "Done When" test criteria are satisfied with host `t.TempDir()` unit tests and, where specified, shell template golden tests; no new host-git test files outside the 34-file allowlist.

### Completion Criteria

- [ ] All tasks in sequence completed successfully
- [ ] Quality verification tasks passed
- [ ] Code review completed and issues addressed
- [ ] Documentation updated

## Task Alignment

| Task | Task Objective | Contribution to Sequence Goal |
|------|----------------|-------------------------------|
| 01_json_flag_standardization | Alias `--json` on `--format` commands; rename `flow add --json`; wrap remaining commands in `jsoncontract.RunE`; fix `status all` empty-case JSON | Eliminates the two-idiom split so every machine consumer gets the same flag and error envelope |
| 02_exit_code_table | Define one CLI-wide exit-code table; align doctor/sync/clone options; replace `os.Exit` calls with typed `CommandError` | Makes exit codes predictable for wrapper scripts and festival-app |
| 03_signal_context | Wire `signal.NotifyContext` at the root; delete nil-ctx fallbacks; fix the retry backoff sleep | Lets Ctrl-C and SIGTERM actually cancel long operations instead of relying on process kill |
| 04_alias_and_usage_fixes | Remove duplicate `reg` alias; set root `SilenceUsage`; fix or delete dead global flags | Stops mutating commands firing on `camp reg` and stops usage dumps on runtime errors |
| 05_readme_and_flow_channel | Fix README install path; make scaffolded templates channel-aware; pin `docs` recipe `BUILD_TAGS` | Eliminates the first-run install failure and the "unknown command" error in every stable-initialized campaign |
| 06_shell_and_nav_fixes | Fix shell wrapper error suppression; add passthrough to go/switch; fix nav stat and cache locking | Stops `cd "$(camp go --print ...)"` landing in a nonexistent directory and hides no errors |
| 07_repair_failures_and_tty_guards | Surface init registry-save and fest-init failures; warn on swallowed cleanups; add TTY guards | Prevents silent init failures and TUI panics on non-TTY invocations |
| 08_pull_push_flag_handling | Stop bare `-p` hijacking git flags; honor `--`; fix the error hint | Lets `camp pull -p origin` pass `-p` to git untouched |

## Dependencies

### Prerequisites (from other sequences)

- **Sequence 01 (gate_restoration)**: both-profile unit gate must be green before this sequence's quality gates are meaningful.
- **Sequence 04 (gate_enforcement)**: `just gate` recipe and both-profile lint enforcement must exist so task verification commands are stable.
- **Sequence 09 (app_contracts), task 08**: the `Profile` build-tag constant in `camp version --json` is required by task 05 to make scaffolded templates channel-aware.
- **Sequence 09 (app_contracts), task 07**: the manifest annotation completeness work in 09.07 must coordinate with task 07 here (the four TTY-guarded commands must end annotated `agent_allowed: false`).

### Provides (to other sequences)

- **A consistent CLI surface**: all downstream sequences and the 004_POLISH verification checklist assume one exit-code table, one JSON flag, and honest help text.
- **Launch-ready docs**: README install path and channel-accurate scaffolded skills are hard blockers for the public launch path.

## Working Directory

Target project: `projects/worktrees/camp/camp-hardening` (relative to campaign root)

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Task 05 depends on Profile constant not yet landed (09.08) | Medium | Medium | Sequence 09 precedes 10 in the dependency graph; verify 09.08 is committed before starting task 05 |
| `SilenceUsage` on root in task 04 breaks a command that legitimately showed usage on error | Low | Medium | `SetFlagErrorFunc` re-enables usage for cobra parse errors only; verify with the no-usage-on-runtime and usage-on-parse tests in task 04 |
| Wrapping commands in `jsoncontract.RunE` changes error output shape for existing callers | Low | Low | camp is not publicly released; schema_version string lets festival-app key on the new envelope explicitly per D004 |
| Shell template changes break existing sourced shells | Low | Medium | Golden tests capture the before/after template output; wrappers remain backward-compatible for the success path |

## Progress Tracking

### Milestones

- [ ] **Milestone 1**: Tasks 01-04 complete (flag unification, exit codes, signal context, alias/usage cleanup); both-profile gate green.
- [ ] **Milestone 2**: Tasks 05-07 complete (docs/channel, shell/nav, repair/TTY); launch-readiness items resolved.
- [ ] **Milestone 3**: Task 08 complete and full sequence gate passes including named tests.

## Quality Gates

### Testing and Verification

- [ ] Both-profile gate: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint` all pass after each task
- [ ] Every `--json` command on failure emits the error envelope to stderr and exits non-zero (task 01)
- [ ] Exit-code table tests via `Execute()` (not `os.Exit`) for doctor/sync/clone failure modes (task 02)
- [ ] Ctrl-C cancellation test on the retry loop; `ExecuteContext` smoke test (task 03)
- [ ] No-duplicate-aliases test across root children; runtime-error no-usage and parse-error with-usage tests (task 04)
- [ ] Stable-profile scaffold produces no flow instructions; dev-profile does (task 05)
- [ ] Shell template golden tests for go/switch wrapper changes (task 06)
- [ ] Non-TTY invocation tests for the four guarded commands (task 07)
- [ ] `camp pull -p origin` passes `-p origin` to git untouched (task 08)

### Code Review

- [ ] Code review conducted
- [ ] Review feedback addressed
- [ ] Standards compliance verified

### Iteration Decision

- [x] Need another iteration? No
- [x] If yes, new tasks created: none
