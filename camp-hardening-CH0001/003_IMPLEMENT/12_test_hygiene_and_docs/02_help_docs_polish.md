---
fest_type: task
fest_id: 02_help_docs_polish.md
fest_name: help_docs_polish
fest_parent: 12_test_hygiene_and_docs
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.805398-06:00
fest_updated: 2026-06-15T16:00:28.340937-06:00
fest_tracking: true
---


# Task: Help and Docs Polish

## Objective

Fix all named help-text inconsistencies and documentation gaps: rename root `promote` to `shelve`, fix the `camp projects` hint, move embedded examples to the cobra `Example` field, add `Long` text to the 22 Short-only commands, fix the `flow migrate` help contradiction, implement the `appendHistory` JSONL stub, regenerate the CLI reference, and cap the release-notes first-release path.

Findings: R35 (UX-2, UX-3, UX-4, UX-5, UX-10), BR-8, BR-9, and the flow history advertisement.

## Requirements

- [x] **Root `promote` direction fixed** (UX-2): `cmd/camp/promote/promote.go` moves workitems INTO dungeon. Rename the command to `shelve` (Use, Short, Example updated) so the verb matches the direction. `intent promote` is a separate command with the opposite direction; it must not be touched.
- [x] **Deprecated `wt` alias redirected** (UX-3): `cmd/camp/worktrees/root.go:9` carries `Aliases: []string{"wt"}` on a deprecated command. Drop it or replace with a redirect to `camp project worktree`.
- [x] **`camp status` real flags** (UX-4): `cmd/camp/status.go` uses `DisableFlagParsing: true`. Register the real flags (`--sub`, `--project`/`-p`, `--short`/`-s`) and keep a `--` passthrough for git flags, or document the raw passthrough in the Use line.
- [x] **Use-string and registration fixes** (UX-5): `cmd/camp/navigation/shortcuts.go:43` has `Use: "add [name] [path] or [project] [name] [path]"` -- replace "or" with `|`. `cmd/camp/shell_init.go` Long help understates what it installs: add a note that it defines the `camp` function itself and installs `cr`, `csw`, `cint`, `cnote`, `cie`.
- [x] **Wrong `camp projects` hint** (UX-10): `cmd/camp/navigation/shortcuts.go:315` has the error message `"project %q not found (run 'camp projects' to see available projects)"`. No `camp projects` command exists. Fix to `camp project list`.
- [x] **Embedded examples to Example field** (UX-10): Three commands embed "Examples:" inside the Long string instead of using the cobra `Example` field, so the styled template renders nothing in the Examples section:
  - `cmd/camp/registry/root.go:21-25`: move the examples block to `Example:`
  - `cmd/camp/promote/promote.go:34-37`: move to `Example:` (do this in the same commit as the rename to `shelve`)
  - `cmd/camp/dungeon/add.go:33-35`: move to `Example:`
- [x] **Long text for 22 Short-only commands** (UX-10): the following commands have `Short:` only and no `Long:`. Add a `Long:` string to each. The Long should explain what the command does, when to use it, and what input it expects. One solid paragraph each is sufficient.
  - `internal/commands/workitem/adopt.go`
  - `internal/commands/workitem/create.go`
  - `internal/commands/workitem/current.go`
  - `internal/commands/workitem/doctor.go`
  - `internal/commands/workitem/link.go`
  - `internal/commands/workitem/links.go`
  - `internal/commands/workitem/resolve.go`
  - `internal/commands/workitem/unlink.go`
  - `internal/commands/workflow/create.go`
  - `internal/commands/workflow/doctor.go`
  - `internal/commands/workflow/list.go`
  - `internal/commands/workflow/shortcut_add.go`
  - `internal/commands/workflow/show.go`
  - `internal/commands/workflow/sync.go`
- [x] **`flow migrate` help contradiction fixed**: `internal/commands/flow/migrate.go:30-35` claims ready/ items move to `dungeon/ready/` but the code moves them to the root directory. Fix the Long help to accurately describe what `runV1ToV2Migrate` actually does. Also: `--force` is advertised in the Long as "skip confirmation prompts" but no confirmation prompt exists in `runV1ToV2Migrate`. Remove the `--force` advertisement from the help (the flag is registered and used in the legacy path, so keep the flag itself; just fix the v1-to-v2 description).
- [x] **`appendHistory` stub implemented or removed**: `internal/workflow/service_move.go:87-91` is a TODO stub returning nil. `TrackHistory: true` is the schema default. `camp flow show` prints "History: enabled (...)". `internal/commands/flow/flow.go:43` advertises `camp flow history` in its Long. The unregistered subcommand and the stub together mean the advertised feature silently does nothing. **Recommendation: implement it.** It is small: a JSONL append of one JSON line per move, following `internal/dungeon/service.go`'s `AppendCrawlLog` as the exact pattern. If implementing is blocked, the alternative is removal of all advertisements (flow.go:43 Long line, flow show History output, TrackHistory schema default set to false). Document which path was taken in a comment at the stub site.
- [x] **CLI reference regenerated** (BR-8): run `just docs` with `BUILD_TAGS=''` pinned (to produce the stable-profile reference). Verify `docs/cli-reference/camp_workitem_priority.md` exists after regeneration. Commit the regenerated docs.
- [x] **README completion category drift** (BR-8): `README.md:335` lists completion categories `p f w a d i wt du cr pi de`. `a` should be `ai` and `ex` is missing. Fix the example to match the code (`README.md:416` and the code at `internal/config/defaults.go` have the correct list).
- [x] **Release-notes first-release cap** (BR-9): `tools/release-notes/main.go:106-113` in `commitSubjects` feeds the entire repo history into the sections when `previousTag` is empty (first release). Cap this: when `previousTag == ""`, either skip the commit-subject sections entirely (emit only the header and a note that this is the first release) or limit to the last N commits. A clean approach is:
  ```go
  func commitSubjects(tag, previousTag string) ([]string, error) {
      if previousTag == "" {
          return nil, nil // first release: no commit-diff sections
      }
      args := []string{"log", "--format=%s", previousTag + ".." + tag}
      return gitLines(args...)
  }
  ```

## Implementation

### Background

All file:line numbers are verified against the worktree at `projects/worktrees/camp/camp-hardening` (review commit `bc2ad1f`). Verify against the worktree before editing; lines may have drifted from prior sequence work.

This task touches many files but each change is small and independent. Work file by file; run the both-profile gate after every 3-5 files to keep the feedback loop tight.

The `docs` recipe in the root `justfile` (line 124) runs `build-camp` with no explicit `BUILD_TAGS`. Pin `BUILD_TAGS=''` explicitly when regenerating so dev-channel commands (quest, flow) are not included in the stable CLI reference:

```bash
BUILD_TAGS='' just docs
```

### Step 1: Rename `camp promote` to `camp shelve`

**File: `cmd/camp/promote/promote.go`**

The `Cmd` variable has `Use: "promote <status>"` and `Short: "Promote the workitem at cwd to a dungeon status"`. The verb "promote" implies moving forward (as `intent promote` does), but this command shelves to dungeon. Rename:

```go
var Cmd = &cobra.Command{
    Use:     "shelve <status>",
    Short:   "Shelve the workitem at cwd to a dungeon status",
    GroupID: "planning",
    Long: `Shelve the directory-style workitem containing the current working
directory to a named dungeon status. Status directories live under the
workitem type's local dungeon (workflow/<type>/dungeon/<status>/); outside
the dungeon a workitem is treated as active.

Run this from anywhere inside workflow/<type>/<slug>/. The workitem
boundary is detected from cwd. The status argument is the destination
directory name (e.g., completed, archived, someday) - no need to spell
out "dungeon/".`,
    Example: `  camp shelve completed   Shelve the workitem to its local dungeon/completed
  camp shelve archived    Move to dungeon/archived
  camp shelve someday     Move to dungeon/someday`,
    ...
}
```

Also update `Aliases` if the command had `promote` as an alias to itself (check for aliases referencing the old name). Update any root.go registration that references the old name.

Search for callers of `camp promote` in tests (`grep -rn '"promote"\|camp promote' tests/ cmd/`). Update test assertions to use `shelve`.

**Why not fold into `dungeon move`:** `dungeon move` operates on arbitrary items by path, while `shelve` detects the workitem boundary from cwd. They are different interfaces. Rename is cleaner than folding.

### Step 2: Deprecated worktrees `wt` alias

**File: `cmd/camp/worktrees/root.go`**

The command is already deprecated. Locate `Aliases: []string{"wt"}` (line 9 area) and remove it:

```go
var Cmd = &cobra.Command{
    Use:        "worktrees",
    Short:      "Manage git worktrees for projects",
    GroupID:    "project",
    Deprecated: "use 'camp project worktree' instead for project-scoped operations",
    // Aliases removed: "wt" now resolves to project/worktree/root.go which already has it
    Long: `...`,
}
```

Verify that `cmd/camp/project/worktree/root.go` already has `Aliases: []string{"wt"}` so `camp wt` resolves to the non-deprecated surface.

### Step 3: camp status real flags

**File: `cmd/camp/status.go`**

Currently `DisableFlagParsing: true` means all flags are parsed by hand in `runStatus`. The fix is to register real flags and pass through to git using `--`:

```go
var (
    statusSub     bool
    statusProject string
    statusShort   bool
)

var statusCmd = &cobra.Command{
    Use:   "status [-- <git-flags>]",
    Short: "Show git status of the campaign",
    Long: `Show git status of the campaign root directory.

Works from anywhere within the campaign - always shows the status
of the campaign root repository.

Pass git flags after -- to forward them directly to git status.`,
    Example: `  camp status              Full status
  camp status --sub        Status of current submodule
  camp status -p camp      Status of the camp project
  camp status -- --short   Short format via git passthrough`,
    RunE: runStatus,
    // DisableFlagParsing removed
}

func init() {
    rootCmd.AddCommand(statusCmd)
    statusCmd.GroupID = "git"
    statusCmd.Flags().BoolVar(&statusSub, "sub", false, "Status of the submodule detected from current directory")
    statusCmd.Flags().StringVarP(&statusProject, "project", "p", "", "Status of a specific project path")
    statusCmd.Flags().BoolVarP(&statusShort, "short", "s", false, "Give output in short format (git --short)")
}
```

Update `runStatus` to read from these flag variables instead of hand-parsing `args`. Arguments after `--` (cobra puts them in `cmd.Flags().Args()` when `--` is used, or capture via `ArbitraryArgs`) are forwarded to the git invocation.

### Step 4: Shortcuts Use-string and hint

**File: `cmd/camp/navigation/shortcuts.go`**

Line ~43:
```go
Use: "add [name] [path] or [project] [name] [path]",
```
Change to:
```go
Use: "add <name> <path> | <project> <name> <path>",
```

Line ~315 (in `findProjectIndex` caller):
```go
return fmt.Errorf("project %q not found (run 'camp projects' to see available projects)", projectName)
```
Change to:
```go
return fmt.Errorf("project %q not found (run 'camp project list' to see available projects)", projectName)
```

### Step 5: shell-init Long text

**File: `cmd/camp/shell_init.go`**

The Long help currently says it provides `cgo`, tab completion, and category shortcuts. It does not mention that it redefines the `camp` shell function itself, or that it installs `cr`, `csw`, `cint`, `cnote`, `cie`. Add this to the Long:

```
IMPORTANT: this defines a shell function named 'camp' that wraps the
camp binary. The function intercepts 'camp switch' and 'camp go' to
perform directory changes in the current shell session.

The following shell aliases and functions are also installed:
  cr     camp run (run a just recipe in a project)
  csw    camp switch (shorthand)
  cint   camp intent add (quick idea capture)
  cnote  camp intent note (add a note to an existing intent)
  cie    camp intent explore (interactive intent browser)
```

### Step 6: Move embedded Examples to Example field

For each command below, move the text that appears under "Examples:" inside the `Long:` string to the `Example:` field instead. Remove the "Examples:" heading from Long.

**`cmd/camp/registry/root.go`** (lines 21-25 area): the Long contains:
```
Examples:
  camp registry prune             Remove entries for non-existent campaigns
  camp registry prune --dry-run   Show what would be removed
  camp registry sync              Update path for current campaign
  camp registry check             Check for issues
```
Move to:
```go
Example: `  camp registry prune             Remove entries for non-existent campaigns
  camp registry prune --dry-run   Show what would be removed
  camp registry sync              Update path for current campaign
  camp registry check             Check for issues`,
```

**`cmd/camp/promote/promote.go`** (now `shelve`): done in Step 1.

**`cmd/camp/dungeon/add.go`** (lines 33-35 area): the Long contains:
```
Examples:
  camp dungeon add          Initialize dungeon (skip existing files)
  camp dungeon add --force  Overwrite existing documentation
```
Move to the `Example:` field, remove from `Long:`.

### Step 7: Long text for Short-only commands

For each of the 14 workitem and workflow commands listed in Requirements, add a `Long:` string to the cobra.Command struct. Each Long should:

1. Describe what the command does.
2. Name the file it reads or writes (e.g., `.workitem` metadata file, `workflow/` directory).
3. Note when to use `--json` if the command supports it.

Example for `workitem create`:
```go
Long: `Create a new workitem directory with minimal v1 metadata.

The workitem is created under workflow/<type>/<slug>/ relative to the
campaign root. A .workitem file is written with the id, type, title, and
creation timestamp. Use --type to pick the workflow collection (default:
feature). Use --id to override the generated slug-based id. Use --json
for machine-readable output including the new workitem id and path.`,
```

Example for `workflow create`:
```go
Long: `Create a new workflow collection directory with a .workflow.yaml schema.

Workflow collections group related workitems under workflow/<name>/.
The schema defines the item lifecycle (status directories) and optional
dungeon routing. Use --type to set the item type for this collection.
Run 'camp flow show' to inspect the schema after creation.`,
```

Write similar Long text for each of the 14 commands. The exact wording is flexible; accuracy and coverage of the three points above are the standard.

### Step 8: flow migrate help contradiction

**File: `internal/commands/flow/migrate.go`**

The Long around lines 28-35 says:
```
For v1→v2 migration:
  - active/ items move to root directory
  - ready/ items move to dungeon/ready/
```

The code in `runV1ToV2Migrate` (via `svc.MigrateV1ToV2`) moves active/ items to the root (correct) and ready/ items to the **root** as well (not `dungeon/ready/`). Fix the Long:

```
For v1→v2 migration:
  - active/ items move to the workflow root directory
  - ready/ items move to the workflow root directory
  - Empty active/ and ready/ directories are removed after migration
  - Schema is updated to version 2
```

Also: remove "Use --force to skip confirmation prompts." from the v1-to-v2 description. The legacy path does use `--force`, but `runV1ToV2Migrate` ignores it. If adding confirmation to v1-to-v2 is desired, that is a separate task; for now just remove the false advertisement.

### Step 9: Implement appendHistory (preferred path)

**File: `internal/workflow/service_move.go`**

The stub at line ~89:
```go
func (s *Service) appendHistory(ctx context.Context, entry HistoryEntry) error {
    // TODO: Implement history file writing
    return nil
}
```

Implement using the dungeon `AppendCrawlLog` pattern as a guide. The history file path is `s.schema.HistoryFile` (a relative path from the workflow root, set in the schema). Append a JSON line per call:

```go
func (s *Service) appendHistory(ctx context.Context, entry HistoryEntry) error {
    if err := ctx.Err(); err != nil {
        return err
    }

    histPath := filepath.Join(s.root, s.schema.HistoryFile)
    if err := os.MkdirAll(filepath.Dir(histPath), 0755); err != nil {
        return camperrors.Wrap(err, "create history directory")
    }

    f, err := os.OpenFile(histPath, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return camperrors.Wrap(err, "open history file")
    }
    defer f.Close()

    line, err := json.Marshal(entry)
    if err != nil {
        return camperrors.Wrap(err, "marshal history entry")
    }
    _, err = fmt.Fprintf(f, "%s\n", line)
    return err
}
```

Add the `encoding/json` import if not already present. Also register the `history` subcommand in `internal/commands/flow/flow.go` (add a minimal `newHistoryCommand()` that reads and pretty-prints the history file). If registering the full subcommand is out of scope for this sequence, at minimum remove the `camp flow history` advertisement from `flow.go:43` until the subcommand is registered.

**Removal path (if implementation is blocked):** Set `TrackHistory` schema default to `false` in `internal/workflow/schema.go`, remove the "History: enabled" output from `flow show` (`internal/commands/flow/show.go:86-87`), and remove the `camp flow history` line from `flow.go:43`. Add a comment at the stub: `// TODO(flow-history): implement JSONL append before advertising; tracked as intent <id>`.

### Step 10: Regenerate CLI reference

Run from the worktree root:
```bash
BUILD_TAGS='' just docs
```

This builds a stable-profile binary and runs `gendocs --output docs/cli-reference --format markdown --single`. Verify:

```bash
ls docs/cli-reference/camp_workitem_priority.md   # must exist
git diff --stat docs/cli-reference/               # shows regenerated files
```

Commit the regenerated docs as a separate commit from the code changes.

### Step 11: README completion category fix

**File: `README.md`**

Line ~335:
```
cgo <TAB>                # Completes categories: p f w a d i wt du cr pi de
```

Fix `a` to `ai` and add `ex`:
```
cgo <TAB>                # Completes categories: p f w a d i wt du cr pi de ex ai
```

Verify against `internal/config/defaults.go` and `README.md:416` for the authoritative list before committing.

### Step 12: Release-notes first-release cap

**File: `tools/release-notes/main.go`**

In `commitSubjects` (around line 111), the `else` branch at the bottom currently runs `git log <tag>` (no range), returning all commits since the beginning of time:

```go
func commitSubjects(tag, previousTag string) ([]string, error) {
    args := []string{"log", "--format=%s"}
    if previousTag != "" {
        args = append(args, previousTag+".."+tag)
    } else {
        args = append(args, tag)   // <-- feeds entire history
    }
    return gitLines(args...)
}
```

Replace the `else` branch:
```go
    } else {
        // First release: no range available. Return empty to avoid
        // dumping the entire repo history into the release notes.
        return nil, nil
    }
```

The caller in `main` already handles `nil` slices correctly (sections are only emitted when non-empty). Add a note in the output for first releases:

In `main()`, after resolving `previousTag`:
```go
if previousTag == "" {
    fmt.Println("First release: no previous tag found; skipping commit-diff sections.")
}
```

### Out of scope

- Standardizing all JSON flags to `--json` CLI-wide (R19, tracked separately).
- Adding `--json` support to the many commands that currently lack it (R19/R20).
- Fixing the `pins`/`pin`/`unpin` three-root-commands layout (UX-5 note; fixing the structural issue is R30 territory).
- Pinning `BUILD_TAGS=''` in the `docs` recipe permanently (BR-5 note; that is a recipe change that can land independently).

## Done When

- [x] Both-profile gate green: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`
- [x] `BUILD_TAGS='' just docs` produces zero diff on a second run
- [x] `docs/cli-reference/camp_workitem_priority.md` exists
- [x] `camp shelve --help` shows correct direction-honest text; `camp promote --help` is `intent promote` only
- [x] `camp wt` resolves to `camp project worktree` (non-deprecated), not the deprecated surface
- [x] `camp status --help` shows `--sub`, `--project`, `--short` flags
- [x] `camp shortcuts add --help` shows `|` not `or` in Use
- [x] `camp project list` is the hint text (not `camp projects`) in shortcuts error message
- [x] All 14 Short-only commands now have `Long:` text
- [x] `camp flow migrate --help` accurately describes v1-to-v2 behavior
- [x] `appendHistory` is either implemented and `camp flow history` is registered, or all advertisements are removed
- [x] `README.md:335` shows correct category list
- [x] `tools/release-notes/main.go` caps first-release path

## Completion Notes

- Renamed root `camp promote` to `camp shelve`; left `camp intent promote` untouched.
- Redirected `camp wt` through the project worktree shortcut path and removed the deprecated root alias.
- Replaced `camp status` ad hoc flag parsing with real cobra flags plus git passthrough.
- Implemented workflow history JSONL append and registered `camp flow history`.
- Regenerated stable-profile CLI reference with `BUILD_TAGS='' just docs` and verified a second docs run is idempotent.
- Validation passed: `git diff --check`; stable/dev `just build`; stable/dev `just test unit`; stable/dev `just lint`; targeted promote integration; targeted help and static text checks.