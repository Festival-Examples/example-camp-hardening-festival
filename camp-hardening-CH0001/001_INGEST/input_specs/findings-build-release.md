# Findings: Build Channels, Release Pipeline, tools/release-notes, and Docs Accuracy

Context honored throughout: dev/rc/stable channels and `//go:build dev` gating are intentional design, including quest being dev-only. Findings below are about inconsistencies in how the design is executed, not the design itself.

**Channel wiring (verified clean at the build level):** dev-gated files are `cmd/camp/quest_register.go`, all 11 files in `cmd/camp/quest/`, `internal/commands/release/register_dev.go`, `cmd/camp/release_profile_dev_test.go`, `internal/quest/tui/{create_model.go,create_model_test.go,styles.go}`. Stable counterparts: `internal/commands/release/register_stable.go` (no-op registerDev), `cmd/camp/release_profile_stable_test.go`. Shared assertions in untagged `cmd/camp/release_profile_test.go` with `devOnlyCommands = []string{"flow", "quest"}` (line 15). Both `go build ./...` and `go build -tags dev ./...` compile.

---

## BR-1. Dev-tagged code is never tested, vetted, or linted by any gate (HIGH)

- `internal/buildutil/tasks/test.go:81-84` runs `go test -count=1 -json -short ...` with no build tags; `tasks.Build` honors `BUILD_TAGS` (`build.go`) but `tasks.Test` does not. `.justfiles/test.just` recipes (`verbose`, `coverage`, `short`, `bench`) all run untagged. `justfile` `vet` and `lint` run untagged. There is no `.golangci.yml` and no `.github/workflows/` directory: **no CI exists at all**.
- Consequence: `release_profile_dev_test.go`, all `cmd/camp/quest` tests, and `internal/quest/tui` tests run only when someone manually types `go test -tags dev`. This is exactly how the manifest regression (TEST-1) shipped unseen, and festival-app bundles DEV-profile binaries, so the untested surface is the one the app actually runs.
- **Recommendation:** make `tasks.Test` honor `BUILD_TAGS` like `tasks.Build`; add `just test dev` or run the suite twice (stable plus dev); stand up minimal CI (build both profiles, vet with both tag sets, unit tests both profiles, lint).

## BR-2. Release recipes publish from an ungated tree (MEDIUM)

- `.justfiles/release.just` recipes `dev`/`rc`/`stable`/`promote-rc`/`promote`: tag, push, generate notes, `gh release create`. No `just test`, no `just lint`, no binary upload before tagging.
- With main currently red (TEST-1), a release tag can be cut from a failing tree today. If artifact distribution is entirely the festival repo's job, nothing in the recipes or docs says so.
- **Recommendation:** gate `git tag` behind at minimum `just test unit` in both profiles; document the division of labor with `projects/festival`.

## BR-3. Cross-platform release builds ship with no version stamping (HIGH)

- `justfile:14` defines a correct `ldflags` variable (Version/Commit/BuildDate into `internal/version`) that is referenced by ZERO recipes (verified by grep; dead).
- `.justfiles/build.just:69-104`: the `linux`, `darwin`, `darwin-arm64`, and `windows` recipes use `go build -ldflags '-s -w'` only. Those binaries report `camp dev (built unknown, commit unknown)`.
- `internal/buildutil/tasks/build.go:17-31` stamps via the `VERSION` env var, but nothing in the release flow sets `VERSION`. `internal/version/version.go` has no `debug.ReadBuildInfo()` fallback, so `go install .../cmd/camp@vX.Y.Z` also reports `dev`.
- Honest summary: only buildutil-built binaries with VERSION manually set ever report a real version.
- **Recommendation:** thread the stamping into the cross-platform recipes (or route them through buildutil with VERSION derived from the git tag); add a ReadBuildInfo fallback in `internal/version`.

## BR-4. The build channel is invisible to machines (MEDIUM)

- There is no Profile/Channel field in `internal/version/version.go:7-26`; a dev-profile and stable-profile binary at the same tag are indistinguishable except via the hidden, text-only `build-profile` command (`internal/commands/release/register.go`).
- festival-app intentionally bundles dev-profile camp and shells out to it; it has no machine-readable way to assert "this binary has quest/flow commands" before failing on unknown command.
- **Recommendation:** add `Profile string` set by a `//go:build dev` / `//go:build !dev` constant pair, surface it in `camp version --json`, document in docs/json-contracts.md.

## BR-5. Channel-boundary inconsistencies (MEDIUM)

- **Stable binaries scaffold quest surfaces with no quest command.** Stable code imports `internal/quest` from `internal/commands/workitem/{link.go:20,ref_quest.go:12,doctor.go:22}`, `internal/scaffold/{init.go:18,repair.go:17}`, and `internal/workitem/links/validate.go:12`; stable `camp init` scaffolds `.campaign/quests/default/quest.yaml.tmpl`. Stable users get quest directories and quest-ref validation but `camp quest` is unknown. Either document as forward-compatible state or gate the scaffolding behind the channel.
- **Two registration mechanisms for the same concern (LOW):** `flow` registers via `release.registerDev` (`register_dev.go:10-15`) while `quest` registers via a main-package `init()` in `cmd/camp/quest_register.go`. Both covered by profile tests; pick one pattern.
- **`docs` recipe can leak dev pages (LOW):** `justfile` `docs` runs `build-camp` honoring ambient `BUILD_TAGS`; run from a dev shell it would commit quest/flow pages into the stable CLI reference. Pin `BUILD_TAGS=''` in the recipe.
- **`windows` build recipe exists (`.justfiles/build.just:99-104`) although Windows is not a supported target per project policy (LOW).** Invites accidental artifact publication.

## BR-6. The flow channel contradiction: scaffolded skills instruct dev-only commands (HIGH)

- `internal/scaffold/campaign/templates/.campaign/skills/campaign-workflows/SKILL.md.tmpl:33-35` instructs agents to run `camp flow status`, `camp flow move`, `camp flow sync`; `templates/dungeon/OBEY.md.tmpl:36-42` instructs `camp flow move ...`. `flow` is dev-only (`register_dev.go`).
- README.md:15 and :206 advertise `camp flow` with no channel caveat. `docs/cli-reference/camp_dungeon_add.md:15` and `camp-reference.md:863` reference `'camp flow init'` in stable docs, and that text comes from the `dungeon add` Long help, so stable binaries themselves display a hint to a command they do not have.
- **Why it matters:** every campaign initialized with a stable binary ships agent skills whose commands return "unknown command". This is a launch-readiness issue given the <5 minute first-run goal.
- **Recommendation:** decide flow's channel. Either promote `flow` to stable (the scaffold clearly assumes it) or make the templates channel-aware and scrub README plus the dungeon add help text.

## BR-7. README install command is broken (HIGH)

- `README.md:28` (verified this session): `go install github.com/Obedience-Corp/camp@latest`. The module root has no main package (main is `./cmd/camp`), so the documented install fails with "is not a main package". Must be `go install github.com/Obedience-Corp/camp/cmd/camp@latest`.
- Direct first-run blocker for the public launch path.

## BR-8. Generated CLI reference is stale (MEDIUM)

- No `docs/cli-reference/camp_workitem_priority.md` exists and `camp-reference.md` has zero mentions of `workitem priority` (added in e7a66e6 / PR #318). `just docs` was not rerun. Same root cause class as TEST-1: no gate forces doc regeneration.
- Minor README drift (LOW): README.md:335 completion example lists categories `p f w a d i wt du cr pi de`; `a` should be `ai` and `ex` is missing (correct at README.md:416 and in code).
- **Recommendation:** regenerate docs; add a CI check that `just docs` produces no diff.

## BR-9. tools/release-notes: wired in, decent quality (LOW)

- Factually wired: `.justfiles/release.just` private `notes` recipe runs `go run ./tools/release-notes --repo ... --tag ... --output dist/release-notes.md`, and every release recipe (dev, rc, stable, promote-rc, promote) calls it before `gh release create --notes-file`. Not orphaned.
- Quality: tag parsing is regex-anchored per channel; previous-tag resolution skips same-commit tags; dedupe and categorization have tests (`tools/release-notes/main_test.go`, `tools/release-notes/internal/notes/notes_test.go`).
- Issues: first-release path (`commitSubjects`, `tools/release-notes/main.go:106-113`) feeds the entire repo history into the sections; cap or skip sections when previousTag is empty. Uses `fmt.Errorf` and no context on git execs; acceptable for a build tool but should be declared an explicit exemption from the error standard.

## BR-10. Docs spot-check results (informational)

Verified correct against code: registry path `~/.obey/campaign/registry.json` and config path (`internal/config/registry.go:24`, `global.go:12`); shortcut table including `wt` and `ex` (`internal/config/defaults.go:82,92`, `internal/nav/shortcuts.go:45-57`); `workitem` flags `--json/--print/--type/--stage/--query/--limit` and aliases `wi`/`workitems` (`internal/commands/workitem/workitem.go:36-130`); `switch --print`, `list --format`, `commit --sub`, status/push/pull `--sub`; `camp create/register/unregister/gather/fresh` all exist; the workitem-commit staging matrix doc matches `commit.go`. The `camp-workitems` skill template is accurate, including its description of `.campaign/settings/workitems.json` as the tool-managed priority store (consistent with `priority.StorePath`).
