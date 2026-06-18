---
fest_type: task
fest_id: 01_json_flag_standardization.md
fest_name: json_flag_standardization
fest_parent: 10_cli_consistency
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.768983-06:00
fest_updated: 2026-06-15T02:09:29.728524-06:00
fest_tracking: true
---


# Task: JSON Flag Standardization

## Objective

Unify JSON output behind a single `--json` flag CLI-wide by aliasing it onto the remaining `--format` commands, renaming the `flow add` input flag to avoid the direction conflict, wrapping the remaining `--json` commands in `jsoncontract.RunE`, and fixing the `status all` empty-case human string that breaks JSON parsers.

## Requirements

- [ ] `--format` commands (`cmd/camp/list.go:58`, `cmd/camp/project/list.go:35`, `cmd/camp/dungeon/list.go:56`) each gain a `--json` bool alias that, when set, forces `format = "json"` before routing to the existing format switch. Note: sequence 09 task 04 already migrated the intent family (`intent/list.go`, `find.go`, `count.go`, `show.go`) to the `intents/v1alpha1` envelope with a native `--json` flag; do not touch those files.
- [ ] `internal/commands/flow/add.go:130` renames the `--json` input flag to `--from-json` (UX-8); the `StringVarP` shorthand `-j` is removed or reassigned to avoid colliding with the new meaning; all references to `flowAddJSON` in the same file are updated accordingly.
- [ ] Every command in the "remaining" set (verified at worktree: `cmd/camp/doctor.go`, `cmd/camp/sync.go`, `cmd/camp/clone.go`, `cmd/camp/status_all.go`, `cmd/camp/quest/list.go`, `cmd/camp/quest/show.go`, `cmd/camp/quest/links.go`, `cmd/camp/worktrees/list.go`, `cmd/camp/worktrees/info.go`, `cmd/camp/skills/status.go`, `cmd/camp/leverage/main_command.go`, `cmd/camp/leverage/history.go`) has its `RunE` wrapped with `jsoncontract.RunE(schemaVersion, func() bool { return <jsonFlag> }, ...)` following the pattern at `internal/commands/workitem/workitem.go:50-52`.
- [ ] `cmd/camp/status_all.go:104-107`: the `fmt.Println(ui.Info("No submodules found in this campaign"))` branch is conditioned so that under `--json` it instead emits `{"schema_version":"status-all/v1alpha1","timestamp":"<RFC3339>","repos":[]}` to stdout and returns; the human string goes to stderr only in non-JSON mode. This satisfies UX-20 and D004's `status-all/v1alpha1` versioning.
- [ ] No new schema versions are introduced beyond `status-all/v1alpha1` for the empty case and using the existing surrounding command schema strings for the `jsoncontract.RunE` wrappers on commands that have existing JSON output shapes (use a placeholder constant like `"doctor/v1alpha1"` for commands that had no formal schema; document it in `docs/json-contracts.md`).
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

Two incompatible idioms exist today. The majority of the CLI uses `--json bool`, but seven commands use `--format string` with `json` as a value (`cmd/camp/list.go:58` with `-f` shorthand, `cmd/camp/project/list.go:35`, `cmd/camp/dungeon/list.go:56`). The intent family had the same split; sequence 09 task 04 converted those to native `--json` with the `intents/v1alpha1` envelope. This task finishes the remaining non-intent `--format` commands and wraps all bare `--json` commands in `jsoncontract.RunE` so errors emit the structured envelope instead of a plain text `Error:` line.

The `flow add --json` flag is an input flag (takes a JSON string to pre-populate the form), which is the opposite direction from every other `--json` in the codebase. Renaming it to `--from-json` is a dev-channel-only change since `flow` is dev-only.

The `status all --json` empty-case bug is verified at `cmd/camp/status_all.go:104-107`: when `len(paths) == 0`, `fmt.Println(ui.Info(...))` runs and `return nil` fires before the `statusAllJSON` branch at line 120. Any downstream JSON parser receives a styled human sentence instead of valid JSON.

Note: verify these line numbers against the worktree before editing; the review cited `5c22d2b`/`11c270f` and the worktree branch is at `bc2ad1f`. Read each file first to confirm current line numbers.

### Verified file:line targets (as of `bc2ad1f`; verify before editing)

- `cmd/camp/list.go:58` - `StringP("format", "f", "table", ...)`
- `cmd/camp/project/list.go:35` - `StringP("format", "f", "table", ...)`
- `cmd/camp/dungeon/list.go:56` - `StringP("format", "f", "table", ...)`
- `internal/commands/flow/add.go:130` - `StringVarP(&flowAddJSON, "json", "j", "", "JSON input...")`
- `cmd/camp/status_all.go:49` - `BoolVar(&statusAllJSON, "json", false, ...)`
- `cmd/camp/status_all.go:104-107` - empty-path human string before `statusAllJSON` branch

### Step-by-step approach

**Step 1: Alias `--json` on the three `--format` commands.**

For each of `list.go`, `project/list.go`, and `dungeon/list.go`, add a `--json` bool flag and wire it to force `format = "json"`:

```go
// In the init() function, after the existing --format registration:
var jsonOutput bool
cmd.Flags().BoolVar(&jsonOutput, "json", false, "Output as JSON (shorthand for --format json)")
```

Then in the RunE body, before the existing format check, add:

```go
if jsonOutput {
    formatStr = "json"
}
```

The exact variable name holding the format string differs per file (`listCmd` uses a local `formatStr`, `projectListCmd` uses `formatStr`, `dungeonListCmd` uses `format`). Read each file to confirm before editing. The key requirement is that `--json` resolves to the same code path as `--format json`.

**Step 2: Rename `flow add --json` to `--from-json`.**

In `internal/commands/flow/add.go`, locate the flag declaration near line 130:

```go
cmd.Flags().StringVarP(&flowAddJSON, "json", "j", "", `JSON input (use "-" for stdin)`)
```

Change to:

```go
cmd.Flags().StringVarP(&flowAddJSON, "from-json", "", "", `JSON input (use "-" for stdin)`)
```

Remove the `-j` shorthand (leave the shorthand parameter empty or omit the `VarP` variant by switching to `StringVar`). The variable name `flowAddJSON` can remain; only the flag name visible to users changes. Update the Long help text on the command (lines around 55-57) to use `--from-json` in the examples.

**Step 3: Wrap remaining `--json` commands in `jsoncontract.RunE`.**

The pattern from `internal/commands/workitem/workitem.go:50-52`:

```go
RunE: jsoncontract.RunE("workitems/v1alpha5", func() bool { return flagJSON }, func(cmd *cobra.Command, args []string) error {
    // ... original RunE body ...
}),
```

For each target command, define a schema version constant (e.g., `const doctorSchemaVersion = "doctor/v1alpha1"`), find the existing `RunE:` assignment, and wrap it. If the command uses a named function instead of an inline function (e.g., `RunE: runDoctor`), convert it to an inline wrapper that calls `runDoctor`:

```go
RunE: jsoncontract.RunE("doctor/v1alpha1", func() bool { return doctorOpts.jsonOutput }, func(cmd *cobra.Command, args []string) error {
    return runDoctor(cmd, args)
}),
```

Commands to wrap and their `--json` variable names (verify flag variable names by reading each file):

| File | JSON flag var | Schema string |
|---|---|---|
| `cmd/camp/doctor.go` | `doctorOpts.jsonOutput` | `"doctor/v1alpha1"` |
| `cmd/camp/sync.go` | `syncOpts.jsonOutput` (verify var name) | `"sync/v1alpha1"` |
| `cmd/camp/clone.go` | `cloneOpts.jsonOutput` (verify var name) | `"clone/v1alpha1"` |
| `cmd/camp/status_all.go` | `statusAllJSON` | `"status-all/v1alpha1"` |
| `cmd/camp/quest/list.go` | local json flag (verify) | `"quest-list/v1alpha1"` |
| `cmd/camp/quest/show.go` | local json flag (verify) | `"quest-show/v1alpha1"` |
| `cmd/camp/quest/links.go` | local json flag (verify) | `"quest-links/v1alpha1"` |
| `cmd/camp/worktrees/list.go` | local json flag (verify) | `"worktrees-list/v1alpha1"` |
| `cmd/camp/worktrees/info.go` | local json flag (verify) | `"worktrees-info/v1alpha1"` |
| `cmd/camp/skills/status.go` | local json flag (verify) | `"skills-status/v1alpha1"` |
| `cmd/camp/leverage/main_command.go` | local json flag (verify) | `"leverage/v1alpha1"` |
| `cmd/camp/leverage/history.go` | local json flag (verify) | `"leverage-history/v1alpha1"` |

After wrapping, `doctor.go`'s existing JSON branch must still call `outputDoctorJSON(result)` for the success path; the wrapper only intercepts errors. Verify that `runDoctor` returns the error from `outputDoctorJSON` on success, which is already the case per the verified code. The `exitDoctorWithCode` call belongs to task 02 (exit code table); do not move it in this task.

**Step 4: Fix the `status all` empty-case.**

In `cmd/camp/status_all.go`, the current code near line 104:

```go
if len(paths) == 0 {
    fmt.Println(ui.Info("No submodules found in this campaign"))
    return nil
}
```

Replace with:

```go
if len(paths) == 0 {
    if statusAllJSON {
        payload := struct {
            SchemaVersion string        `json:"schema_version"`
            Timestamp     time.Time     `json:"timestamp"`
            Repos         []repoStatus  `json:"repos"`
        }{
            SchemaVersion: "status-all/v1alpha1",
            Timestamp:     time.Now().UTC(),
            Repos:         []repoStatus{},
        }
        enc := json.NewEncoder(os.Stdout)
        enc.SetIndent("", "  ")
        return enc.Encode(payload)
    }
    fmt.Fprintln(os.Stderr, ui.Info("No submodules found in this campaign"))
    return nil
}
```

Import `"time"` and `"encoding/json"` if not already present (both are already imported in this file per the review).

**Step 5: Update `docs/json-contracts.md`.**

Add a row for each new `v1alpha1` schema string with a description and a note that these are initial envelopes with no guaranteed stability yet.

### Edge cases

- `cmd/camp/sync.go` and `cmd/camp/clone.go` use `os.Exit` inside RunE today (task 02 will fix that). In this task, wrap the RunE anyway; the `os.Exit` path currently bypasses the wrapper, which is acceptable as a transient state. Task 02 will replace the `os.Exit` calls and the wrapper will then intercept failures correctly.
- The quest commands are dev-gated (`//go:build dev`). The `jsoncontract.RunE` wrapper does not require a build tag; include it unconditionally within the dev-tagged file.
- The `--format` commands expose a `-f` shorthand. The new `--json` alias does not conflict with `-f`; both flags will work. Do not remove `-f`.

### Out of scope

- Sequence 09 task 04 already handled intent list/find/count/show; do not touch those files.
- UX-2/3/5/10 help text polish is sequence 12.02.
- Flow items `--json` dead flag (N-7) is sequence 09 task 02.
- Status all cache write (N-18) is sequence 08 task 04.

## Done When

- [ ] All requirements met
- [ ] `camp list --json` and `camp list --format json` produce identical output
- [ ] `camp project list --json` and `camp dungeon list --json` similarly work
- [ ] `internal/commands/flow/add.go` flag is named `--from-json` with no `-j` shorthand
- [ ] `camp status all --json` with zero submodules emits valid JSON with `"schema_version":"status-all/v1alpha1"` and an empty `"repos":[]`; the JSON is parseable by `jq`
- [ ] Running `camp doctor --json` when a check fails exits non-zero and emits the `jsoncontract` error envelope to stderr, not a plain `Error:` line
- [ ] Same pattern confirmed for `camp sync`, `camp clone`, and a representative quest/worktrees/skills/leverage command
- [ ] Both-profile gate passes: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`