# Findings: Command Surface and UX Consistency

**Scope:** ~160 commands across 9 help groups (setup, campaign, git, navigation, registry, project, planning, global, system). Roots registered in `cmd/camp/root.go:196-224` plus per-file `init()` functions; `release.Register` adds `fresh` (and `flow` plus `profile` under dev via `internal/commands/release/register_dev.go:14`). Dev-gated: `quest` (`cmd/camp/quest_register.go`), `flow`. Hidden: `complete` (`cmd/camp/complete.go:11`), `__manifest` (`cmd/camp/manifest.go:26`), `gendocs` (`cmd/camp/gendocs.go:25`), plus hidden flag `workitem --path-output` (`internal/commands/workitem/workitem.go:124`).

Shell aliases installed by `shell-init` (parity confirmed across zsh/bash/fish templates in `internal/shell/templates/`): a `camp()` wrapper function that intercepts `switch/sw` and `go/g` to actually cd, plus `cgo`, `cr` (camp run), `csw`, `cint` (intent add), `cnote` (intent note), `cie` (intent explore).

Live-binary verification: findings UX-1 (`camp reg`), UX-12 (usage dump on runtime error), and UX-4 (`camp status --help` showing only `-h`) were confirmed by running the installed dev binary. The doctor JSON exit-0 path (UX-11) was re-verified by direct read of `cmd/camp/doctor.go:114-121` in this session.

---

## Naming and registration

### UX-1. Alias collision: `camp reg` is ambiguous and the mutating command wins (HIGH)
- `cmd/camp/register.go:36`: `Aliases: []string{"reg"}` on `register`.
- `cmd/camp/registry/root.go:10`: `Aliases: []string{"reg"}` on `registry`.
- Verified live: `camp reg` executed **register** and immediately wrote to the registry ("Registered: obey-campaign"). Which command wins depends on cobra registration order, which is fragile.
- **Why it matters:** a user or agent typing `reg` expecting the read-mostly `registry` family triggers a mutating side effect with no confirmation.
- **Recommendation:** remove `reg` from one command (suggest keeping it on `registry`). Add a test asserting no duplicate aliases among root children.

### UX-2. Two opposite-direction commands both named `promote` (MEDIUM)
- `cmd/camp/promote/promote.go:21-22`: `promote <status>` moves the cwd workitem INTO `dungeon/completed|archived|someday` (shelving).
- `cmd/camp/intent/promote.go:21-22`: `promote <id>` advances an intent FORWARD through the pipeline.
- Same verb, inverted semantics, both in the `planning` group. `dungeon move` already covers the shelving concept.
- **Recommendation:** rename root `promote` to something direction-honest (`camp shelve <status>`) or fold into `dungeon move`; at minimum rewrite the Short text.

### UX-3. Deprecated `worktrees` keeps the prime `wt` alias (LOW)
- `cmd/camp/worktrees/root.go:9-10`: `Deprecated: "use 'camp project worktree' instead..."` yet `Aliases: []string{"wt"}`. The replacement `project worktree` also uses `wt` (`cmd/camp/project/worktree/root.go:9`), but `camp wt` resolves to the deprecated surface.
- **Recommendation:** drop the root-level `wt` alias or dispatch it to `project worktree`.

### UX-4. `camp status` documents flags it never registers (LOW)
- `cmd/camp/status.go:16-34`: `DisableFlagParsing: true`; the Long advertises `--sub`, `-p/--project`, `-s/--short`, parsed by hand in `runStatus` (status.go:49-53). Verified live: `camp status --help` shows only `-h, --help`. Completion never offers the real flags; global flags get passed to git silently.
- **Recommendation:** register real flags and use `--` passthrough for git flags, or document the passthrough in the Use line.

### UX-5. Minor Use-string and registration inconsistencies (LOW)
- `cmd/camp/navigation/shortcuts.go:43`: `Use: "add [name] [path] or [project] [name] [path]"` embeds the literal word "or"; should be `add <name> <path> | <project> <name> <path>`.
- `cmd/camp/pins.go:61-63`: `pins`, `pin`, `unpin` as three root commands, one letter apart, with no `pin list|add|rm` canonical tree.
- `cmd/camp/shell_init.go:14-30` understates what shell-init installs: the generated script redefines `camp` itself as a shell function (`internal/shell/templates/zsh.sh.tmpl:11`) and installs `cr`, `csw`, `cint`, `cnote`, `cie`; the help mentions none of this. Users should be told before eval-ing.

---

## Flags

### UX-6. Two dead global flags advertised on every command (HIGH)
- `cmd/camp/root.go:192`: `--config` binds `cfgFile`, never read anywhere (grep confirms only declaration plus bind).
- `cmd/camp/root.go:194`: `--verbose` binds a package var with no consumers; only `flow sync` reads the persistent flag directly (`internal/commands/flow/sync.go:59`). Four commands define their own local `--verbose -v` that shadows the global: `cmd/camp/sync.go:77`, `cmd/camp/clone.go:105`, `cmd/camp/doctor.go:70`, `cmd/camp/leverage/main_command.go:21`.
- **Why it matters:** every help screen advertises no-op flags; agents pass `--verbose` and get nothing. Erodes trust in the whole help surface.
- **Recommendation:** delete `--config` or wire it; make persistent `--verbose` feed a shared option and remove the local shadows, or delete it.

### UX-7. JSON output split across two incompatible idioms (MEDIUM)
- `--format` string flags (json as enum value): `cmd/camp/list.go:58` (default `table`, shorthand `-f`), `cmd/camp/project/list.go:35`, `cmd/camp/intent/list.go:46`, `cmd/camp/intent/find.go:45`, `cmd/camp/intent/count.go:34`, `cmd/camp/dungeon/list.go:56`, `cmd/camp/intent/show.go:45`.
- `--json` bool flags: `cmd/camp/version.go:51`, `cmd/camp/sync.go:83`, `cmd/camp/doctor.go:72`, `cmd/camp/clone.go:107`, `cmd/camp/root_path.go:31`, `cmd/camp/status_all.go:49`, `cmd/camp/quest/{list.go:31,show.go:33,links.go:37}`, `cmd/camp/worktrees/{list.go:53,info.go:50}`, `cmd/camp/skills/status.go:37`, `cmd/camp/leverage/{main_command.go:17,history.go:41}`, all of `internal/commands/workflow/*` and `internal/commands/workitem/*`, `internal/commands/flow/items.go:84`.
- So `camp list --format json` but `camp status all --json`; `camp intent list -f json` but `camp workitem --json`.
- **Recommendation:** standardize on `--json` (the majority, and the agent-facing trees already use it); keep `--format` only where multiple non-JSON formats genuinely exist and accept `--json` as an alias there.

### UX-8. `flow add --json` means JSON INPUT while sibling `--json` means output (MEDIUM)
- `internal/commands/flow/add.go:130`: `StringVarP(&flowAddJSON, "json", "j", "", "JSON input (use \"-\" for stdin)")`.
- `internal/commands/flow/items.go:84`: `BoolVar(&flowItemsJSON, "json", false, "output as JSON")`.
- Same flag name, same command family, opposite data direction.
- **Recommendation:** rename the input flag to `--from-json` or accept `--stdin`.

### UX-9. Shorthand and help-string drift (LOW)
- `-f` means `--format` on `list.go:58`, `date.go:47`, `intent/list.go:46` but `--force` on `cmd/camp/dungeon/add.go:44` and `internal/commands/flow/add.go:127`. Reserve `-f` for one meaning CLI-wide.
- `intent show` (`cmd/camp/intent/show.go:45`) is the only command defaulting `--format` to `text` and the only one offering yaml.
- JSON flag help strings vary: "output as JSON" (`version.go:51`), "Output as JSON" (`status_all.go:49`), "Emit JSON output" (`quest/list.go:31`), "emit a structured JSON result" (workitem family), "Output results as JSON for scripting" (`sync.go:83`). Pick one sentence.

---

## Help text

### UX-10. Help-content gaps and one wrong hint (MEDIUM)
- Example coverage is 14 of ~169 commands (~8%). High-traffic agent-facing commands with no Example: `intent add` (`cmd/camp/intent/add.go:33`), `project add` (`cmd/camp/project/add.go:23`), `workitem create` (`internal/commands/workitem/create.go:29`), all `workflow` and `flow` subcommands. Several commands embed "Examples:" inside Long instead of the `Example` field (`registry/root.go:21-25`, `promote/promote.go:34-37`, `dungeon/add.go:33-35`), so the styled usage template's Examples section renders nothing.
- 22 commands have Short only and no Long, concentrated in `internal/commands/workflow/*` and `internal/commands/workitem/*` (create.go:30, adopt.go:21, link.go:35, links.go:23, unlink.go:26, current.go:22, resolve.go:27, doctor.go:70), exactly the planning surface agents drive blind.
- Wrong hint: `cmd/camp/navigation/shortcuts.go:315` says "run 'camp projects' to see available projects" but no `camp projects` command or alias exists (`cmd/camp/project/root.go:13-15` has no Aliases). Following the hint yields "unknown command" or accidental plugin dispatch to a `camp-projects` binary per `root.go:75-80`.
- **Recommendation:** fix the hint to `camp project list` (or add a `projects` alias); move embedded examples to the Example field; add Long text to the workitem/workflow trees first.

---

## Exit codes

### UX-11. `camp doctor --json` exits 0 even when checks fail (HIGH)
- `cmd/camp/doctor.go:114-121` (re-verified this session):

```go
if doctorOpts.jsonOutput {
    return outputDoctorJSON(result)
}
outputDoctorText(result, doctorOpts.verbose, doctorOpts.fix)
return exitDoctorWithCode(result)
```

- `exitDoctorWithCode` (doctor.go:221) is only reached in text mode. In JSON mode the only possible non-zero exit is a JSON encoding error.
- **Why it matters:** `--json` is the agent path. An agent checking the exit code after `camp doctor --json` concludes the campaign is healthy when it is not. This inverts the command's purpose for its primary machine consumer.
- **Recommendation:** apply `exitDoctorWithCode(result)` after JSON output too, or return a typed `CommandError`.

### UX-12. Runtime errors dump full usage text; usage and runtime failures are indistinguishable (HIGH)
- `cmd/camp/root.go:44` sets `SilenceErrors: true` but not `SilenceUsage` (verified: the only Silence setting in root.go).
- Verified live: `camp intent show does-not-exist-xyz` prints ~12 lines of usage and flags before `Error: intent not found: ...`, exit 1, identical to a misspelled flag.
- The problem is already patched piecemeal: `cmd/camp/id.go:17`, `cmd/camp/root_path.go:26`, `cmd/camp/skills/status.go:27`, `internal/commands/workflow/doctor.go:72`, `internal/commands/workitem/doctor.go:75` set `SilenceUsage: true` individually; `cmd/camp/pull.go:75` flips it at runtime after flag parse.
- **Recommendation:** set `SilenceUsage: true` on the root (inherited); keep usage display for genuine parse errors via `SetFlagErrorFunc`; consider exit 2 for usage errors.

### UX-13. Exit code numbers collide with contradictory meanings (MEDIUM)
- `internal/doctor/options.go:142-148`: 1 = warnings, 2 = failures, 3 = partial fix.
- `internal/sync/options.go:182-188`: 1 = preflight failed, 2 = sync failed, 3 = validation failed.
- `internal/clone/options.go:249-255`: 1 = clone failed, 2 = partial SUCCESS, 3 = validation failed.
- Everything else: 1 = any failure (`cmd/camp/main.go:20`), including usage errors.
- Exit 2 means "hard failure" for sync, "failure" for doctor, "mostly worked" for clone. A wrapper script cannot write a generic handler.
- **Recommendation:** one CLI-wide exit code table (0 ok, 1 runtime failure, 2 usage error, 3+ documented command-specific partial states); align the three packages.

### UX-14. Three commands bypass RunE with `os.Exit` (MEDIUM)
- `cmd/camp/sync.go:144-146`, `cmd/camp/doctor.go:223-232`, `cmd/camp/clone.go:153,161,311,317,321`. Everything else propagates through RunE to `cmd/camp/main.go:11-21`, which already maps `camperrors.CommandError.ExitCode` to the process exit code.
- `os.Exit` inside RunE skips deferred cleanup and makes these commands untestable through `Execute()`.
- **Recommendation:** return `camperrors.CommandError{ExitCode: N}` and delete the `os.Exit` calls.

### UX-15. `camp version` uses Run not RunE and swallows marshal errors (LOW)
- `cmd/camp/version.go:20,31-36`: marshal failure prints to stderr and exits 0. Only command in the CLI using `Run`. Convert to RunE.

---

## Error message quality

(See findings-error-handling-standards.md for the fmt.Errorf compliance sweep. These are UX-specific.)

### UX-16. Backwards double-wrap antipattern (LOW)
- `cmd/camp/pins.go:199`: `camperrors.Wrapf(fmt.Errorf("outside campaign root"), "pin path %q (...)", ...)` renders the cause after the hint.
- `cmd/camp/leverage/dir.go:42` and `dir.go:162`: same pattern with synthetic inner errors.
- **Recommendation:** use `camperrors.Newf(...)` when there is no underlying error.

### UX-17. Multiline errors with hardcoded alignment padding (LOW)
- `cmd/camp/switch.go:87`: `"campaign name required in non-interactive mode\n       Usage: camp switch <name> --print"`. The seven spaces assume the `Error: ` prefix width from `main.go:19`. Same pattern in `cmd/camp/init/command.go:166,239`, `cmd/camp/quest/create.go:74-77`, `internal/commands/workitem/workitem.go:63`, `internal/commands/flow/add.go:163-168`, `cmd/camp/intent/add.go:163`.
- **Recommendation:** standardize on single-line parenthetical hints like `(use 'camp init' to create one)` which half the CLI already uses.

### UX-18. Inconsistent remediation hints and lost child errors (LOW)
- Not-found errors sometimes hint the list command (`pins.go:284`, `skills/status.go:61`) and sometimes do not (`cmd/camp/intent/show.go:76`).
- `cmd/camp/pull.go:211` and `cmd/camp/push.go:227` summarize as `"%d repo(s) failed to ..."` with per-repo detail only on stdout earlier.
- `cmd/camp/status.go:75`: `return gitCmd.Run()` surfaces as bare `Error: exit status 1` and loses git's exit code, unlike `pull.go:119-123` which classifies into `CommandError`.

---

## TTY and stream discipline

### UX-19. Four interactive commands launch TUIs with no TTY guard (MEDIUM)
- `intent explore`: `cmd/camp/intent/explore.go:121-122` calls `tea.NewProgram(model, tea.WithAltScreen())` with no IsTerminal check in the file.
- `settings`: `cmd/camp/settings.go:43-69` builds a huh form unconditionally.
- `intent crawl` (`cmd/camp/intent/crawl.go`) and `dungeon crawl` (`cmd/camp/dungeon/crawl.go`): no IsTerminal reference in either command file (internal pickers check stdout at `internal/crawl/picker.go:177` and `internal/dungeon/docs_browser.go:687`, but a fully non-TTY invocation surfaces raw bubbletea/huh errors like "could not open a new TTY").
- The good pattern already exists: `cmd/camp/intent/add.go:162-164`, `cmd/camp/switch.go:86-88`, `cmd/camp/quest/create.go:85`, `internal/commands/flow/add.go:158-168`, `internal/commands/workitem/workitem.go:62-64` all degrade with explicit messages.
- **Recommendation:** add the same guard preamble to the four unguarded commands; ensure they carry `agent_allowed: false` annotations (the annotation system exists at `cmd/camp/shell_init.go:41`, `internal/commands/workitem/workitem.go:51`).
- Related (LOW): TTY detection is split between stdin checks (`internal/nav/tui/keybindings.go:38`) and stdout checks (`internal/ui/theme/forms.go:18`, `internal/crawl/picker.go:177`, `internal/dungeon/docs_browser.go:687`). One helper checking both, used everywhere.

### UX-20. `camp status all --json` emits a human string in the empty case (MEDIUM)
- `cmd/camp/status_all.go:104-107`: when no submodules exist, `fmt.Println(ui.Info("No submodules found in this campaign"))` runs before the `statusAllJSON` branch at line 122. With `--json`, stdout gets a styled sentence and exit 0; a JSON parser downstream chokes.
- **Recommendation:** emit `{"timestamp": ..., "repos": []}` under `--json`; send the human notice to stderr otherwise.

### UX-21. The JSON error contract exists only for workitem/workflow (MEDIUM)
- `internal/jsoncontract` is consumed exclusively by `internal/commands/workitem/*` and `internal/commands/workflow/*` (19 files; zero usage in `cmd/camp/`). Every other `--json` command that fails mid-run prints plain text `Error: ...` to stderr and nothing on stdout.
- **Why it matters:** agents get a different failure shape depending on which subtree they call. festival-app parses both families.
- **Recommendation:** adopt `jsoncontract.RunE` for the remaining `--json` commands (doctor, sync, clone, status all, quest, worktrees, skills, leverage), or document the limitation in docs/json-contracts.md.

### UX-22. Human-mode diagnostics on stdout (LOW)
- `cmd/camp/pull.go` and `cmd/camp/push.go:200-224` print per-repo error lines via `fmt.Println`; `cmd/camp/doctor.go:199` and `cmd/camp/clone.go:290` likewise. `status.go:61` shows the correct stderr pattern. The root cause is that `internal/ui/text.go:23-38` helpers return strings and leave stream choice to call sites.
- **Recommendation:** stream-aware ui helpers plus a lint for `fmt.Println(ui.Error`.

---

## Verified accurate (no action)

`sync`, `stage`, `commit` (workitem trailer behavior matches `commit_workitem.go`), `mv` vs `copy`, `fresh`, `dungeon add`, and `promote` Long texts all match their RunE behavior. A suspected "failed to pull" copy-paste in push.go was checked and is false: `cmd/camp/push.go:227` correctly says push.
