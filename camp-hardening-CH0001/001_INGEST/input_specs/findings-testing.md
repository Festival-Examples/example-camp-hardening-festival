# Findings: Test Suite

**Baseline:** `go vet ./...` clean. A fast unit subset (`internal/config`, `internal/workitem/...`, `internal/scaffold`, `internal/campaign`) ran green in ~12s with `-short -count=1`. The full suite was not run for this review. The 2026-05-29 exploration at `workflow/explore/camp-test-suite-performance-2026-05-29/` was used as baseline context; status of its recommendations is in TEST-9.

**Inventory:** 364 test files, 3,023 test functions; 345 of those are container integration tests under `tests/integration/` (47 files), leaving ~2,678 host unit test functions across ~80k lines. Table-driven style in 160 files. `t.Parallel()` appears only 26 times repo-wide; the host unit suite is effectively parallel-free at the test level, compensated by per-package process parallelism in buildutil (`internal/buildutil/tasks/test.go:60-121`).

---

## TEST-1. The unit test gate is RED on main right now (CRITICAL)

Verified by running it in this session:

```
$ go test -count=1 -short -run TestManifestCommand ./cmd/camp
--- FAIL: TestManifestCommand_AllRestrictedCommandsPresent (manifest_test.go:162: expected exactly 27 restricted commands, got 28)
--- FAIL: TestManifestCommand_AllCommandsHaveAnnotations (manifest_test.go:222: command "workitem priority" is agent_allowed=true but not in allowlist)
```

Same failures under `-tags dev` (43 vs 44). Cause: `camp workitem priority` (PR #318, `internal/commands/workitem/priority.go:38-41`, `agent_allowed: "true"`) was never added to the allowlist and counts in `cmd/camp/manifest_test.go` (~line 150 count block, ~line 209 allowlist block). The PR merged without the repo gate running.

**Why it matters:** `just test unit` and `just build full` fail for everyone; any CI added later starts red; it proves no gate ran on merge (see findings-build-release.md BR-2 for the structural cause: no CI exists).
**Recommendation:** add `workitem priority` to the allowlist, bump the workitem count, regenerate the CLI reference docs (also stale, BR-8).

## TEST-2. Five integration-tagged host test files are dead and one no longer compiles (CRITICAL)

- Files carrying `//go:build integration` outside the container suite: `cmd/camp/commit_integration_test.go` (10 tests), `cmd/camp/pull_integration_test.go` (11), `cmd/camp/status_integration_test.go` (2), `cmd/camp/refs/commands_integration_test.go` (5), `internal/git/lock_integration_test.go` (2).
- No runner ever executes them: the unit runner never passes the tag (`internal/buildutil/tasks/test.go:84`), the integration runner targets only `tests/integration` (`internal/buildutil/tasks/integration.go:341-358`), and there are no GitHub workflows.
- Proof of rot: `go vet -tags=integration ./cmd/camp/...` fails with `cmd/camp/commit_integration_test.go:317:12: undefined: syncParentRef`. The function was removed when submodule sync was pulled from `camp p commit`; the file has not compiled since around 2026-03-13 (commit 752f0b7).
- **Why it matters:** these files create the illusion of commit/pull/refs coverage on the single most user-trusted flow, and a compile-broken tagged file poisons any future `-tags=integration ./...` run.
- **Recommendation:** migrate each into `tests/integration/` (container equivalents partially exist: `workitem_commit_test.go`, `pull_test.go`) or delete; add `go vet -tags=integration ./...` to `just lint` so tagged code cannot rot invisibly.

## TEST-3. Container pool Reset() does not clean the paths tests actually use (HIGH)

- `tests/integration/helpers_container.go:21-35`: Reset removes only `/test`, `/campaigns`, `/root/.config/camp`, `/root/.camp`. But the registry now lives at `/root/.obey/campaign/registry.json` (`tests/integration/create_test.go:15`, `registry_test.go:45`, `helpers_container.go:295`) and many tests work under fixed `/tmp/...` paths (`create_test.go:27` uses `/tmp/create-happy`).
- Direct evidence the leak is known: `tests/integration/settings_test.go:16` manually runs `rm -rf /root/.obey/campaign` before asserting.
- The `/root/.config/camp` path Reset does clean appears to be the pre-migration config location; Reset was never updated for the `.obey` move.
- **Why it matters:** with a pool of 2-6 shared containers plus `t.Parallel()` at the `GetSharedContainer` chokepoint (`tests/integration/main_test.go:114-131`), which test inherits a dirty container is scheduling-dependent. Any test asserting registry absence or exact contents is a latent flake.
- **Recommendation:** add `/root/.obey` and a per-test work root to the Reset rm list; delete the manual cleanup in settings_test.go; drop the legacy `/root/.config/camp` entry.

## TEST-4. Zero-test packages, including code that deletes branches remotely (HIGH)

18 of 96 packages have no test files: `cmd/camp/{attach,cache,cmdutil,refs,registry,skills,worktrees}`, `cmd/camp/project/{linked,worktree}`, `internal/buildutil{,/tasks,/ui}`, `internal/commands/{flow,fresh}`, `internal/config/registryfile`, `internal/dungeon/scaffold`, `internal/prune`, `internal/workitem/selector`.

Ranked by danger:

1. **`internal/prune` (618 lines, zero tests, HIGH).** `deleteLocalBranches` (`prune.go:148`), `deleteRemoteBranches` (`prune.go:153`), `deleteDetachedWorktrees` (`prune.go:143`). This code deletes local branches, pushes remote branch deletions, and removes worktrees, and no test anywhere (unit or container) invokes `camp project prune`. The only adjacent coverage is `tests/integration/fresh_test.go`. Recommendation: table-driven unit tests of the candidate/forced/merged classification with a fake executor, plus a container test against a bare origin including `--remote-delete`.
2. **`cmd/camp/worktrees` (zero tests, HIGH in combination with GIT-1).** `worktrees clean` contains the unconditional `os.RemoveAll` data-loss path (findings-subsystems.md GIT-1) and nothing tests it.
3. **`internal/commands/flow` (~1,100 lines, 11 files, MEDIUM).** Dev-channel only, but festival-app bundles dev-profile camp, so this ships to the app with zero tests. Add container tests for `flow move/migrate/sync`.
4. **`internal/workitem/selector` (145 lines, MEDIUM).** The documented "explicit and stable" selector resolution order is untested; consumed by `link.go` and the new `priority.go`. A silent ordering regression changes which workitem a command mutates. Pure logic, trivially table-testable.
5. `internal/config/registryfile` (LOW): `Path()` env-precedence (`registryfile.go:25-34`) deserves a small table test; the rest of registry behavior is well covered (`internal/config/registry_test.go`, 1,110 lines, plus 13 container tests).

Well-covered hot spots for contrast: scaffold repair (`internal/scaffold/repair_test.go` 885 lines plus 10 container tests), dungeon moves (14 unit files plus 34 integration tests), registry writes (atomicity and mode preservation tested).

## TEST-5. Host-vs-container isolation: current factual state (MEDIUM)

- The container harness is real and well built: `//go:build integration` (49 files), testcontainers-go pool of 2-6 alpine containers built once in TestMain (`tests/integration/main_test.go:37-105`), reset-on-checkout and reset-on-return, git identity configured inside the container (`helpers.go:181-186`). The justfile only applies `-tags=integration` to `./tests/integration/...` (`.justfiles/test.just:67,80`).
- 34 files OUTSIDE `tests/integration` still shell out to real git on the host (heaviest: `internal/git/commit/commit_test.go` 105 call sites, `internal/git/commit_test.go` 36, `internal/doctor/checks/orphan_test.go` 24). All sampled usages stay inside `t.TempDir()` (841 uses across 156 files); HOME overrides use `t.Setenv`; registry paths are isolated via `CAMP_REGISTRY_PATH`/`XDG_CONFIG_HOME`. 136 `os.Chdir` sites restore cwd via cleanup in the files sampled.
- A ratchet gate exists: `lint-no-host-fs-tests` in the justfile blocks NEW host-git test files against a 34-file allowlist (tracked under CW0003-tests-01). The allowlist has not shrunk since it was introduced.
- **Net:** the containerization standard is enforced for new tests but the 34-file legacy backlog is static. Recommendation: schedule a burn-down, starting with `internal/git/commit/commit_test.go` since it is the largest and tests the most safety-critical flows.

## TEST-6. Flakiness risks (MEDIUM)

- Sleep-race lock tests: `internal/git/commit_test.go:940` and `internal/git/retry_test.go:30` release a lock from a goroutine after fixed sleeps (300ms / 150ms) and need the retry loop to win the race. The unit runner's own comment admits contention pressure: per-package timeout was raised from 30s to 120s (`internal/buildutil/tasks/test.go:78-84`). Most likely unit flakes on a loaded machine. Fix: event-driven release (channel) instead of wall clock.
- mtime ticks: 10+ tests sleep 1-2ms to force distinct timestamps (`internal/config/campaign_test.go:341`, `internal/config/registry_test.go:400`, `internal/project/add_test.go:168`, `internal/workflow/schema_test.go:277`, `internal/nav/index/cache_test.go:490`). Prefer `os.Chtimes`.
- Network on cold integration runs: `apk add git` per pooled container (up to 6 fetches per run, `tests/integration/helpers.go:177`) and scc via `go get ...@latest` (`helpers.go:328-334`; the @latest choice is deliberate per project policy). Offline runs cannot start the suite; a prebaked image would remove the startup tax.
- No real network in unit tests (all github URLs are string fixtures); no dependence on global git identity in sampled host tests (31 files set user.name/email per repo).
- `-short` is nearly inert: only 5 files check `testing.Short()`.

## TEST-7. Error-path and cancellation coverage is a strength in tested packages (informational)

Sampled counts (error assertions / cancellation-test lines): `internal/config` 53/38, `internal/intent` 145/36, `internal/git` 56/72, `internal/workitem` 16/14, `internal/scaffold` 15/12. Context cancellation is tested as a first-class concern, consistent with the campaign standard. The gap is the zero-test packages (TEST-4), not depth in covered ones.

## TEST-8. Quality smells (LOW-MEDIUM)

- 363 `Contains(t, output, ...)` assertions in `tests/integration/` on human-readable strings (e.g. "Committed changes to git" at `project_linked_test.go:28`, "Campaign Initialized" at `create_test.go:37`). Copy edits will break dozens of tests at once. Where commands support `--json`, assert on the envelope (the suite already has `json_error_envelope_test.go`).
- `internal/editor/editor_test.go:17-273`: ~17 manual `os.Setenv` save/restore pairs instead of `t.Setenv`; parallel-hostile.
- Zero uses of `t.Chdir` despite go 1.25.6; hand-rolled `chdirForTest` helpers (`cmd/camp/id_test.go:80`) could be replaced.
- Double reset per integration test: `GetSharedContainer` resets on checkout (`main_test.go:122`) and again on return (`main_test.go:131`); the return-side reset is redundant with the next checkout's.
- Good signs: no exact `err.Error()` string matching, golden testdata in 5 packages, benign duplicate test names.

## TEST-9. Status of the 2026-05-29 performance exploration recommendations

| Rec | Status | Evidence |
|---|---|---|
| R2 container pool + t.Parallel | Done | commit de0286f; `main_test.go:37-142`; 489s to ~167s |
| R5 parallel unit packages | Done | commit 47cb587; worker pool `test.go:60-121`; fail-closed package failures (28fc579) |
| R1 tmpfs + drop sync | Closed negative | Reset still has `sync`, as the exploration concluded |
| R3 batch multi-exec helpers | Not done | `RunCampSplit` still multi-exec (`helpers_container.go:265`), `InitCampaign` still 2 execs (`:101`) |
| R4 pin scc / bake image | Not done | scc @latest is a documented deliberate choice (`helpers.go:304-313`); `apk add git` still per container (`helpers.go:177`) |
| R6 drop -count=1 locally | Not done | `test.go:84` still passes `-count=1` |
| R7 migrate host-fs tests to containers | Ratchet only | allowlist still 34 files; `lint-no-host-fs-tests` blocks new violators (CW0003-tests-01) |
