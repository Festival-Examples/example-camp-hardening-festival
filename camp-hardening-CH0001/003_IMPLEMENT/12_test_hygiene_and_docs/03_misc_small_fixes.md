---
fest_type: task
fest_id: 03_misc_small_fixes.md
fest_name: misc_small_fixes
fest_parent: 12_test_hygiene_and_docs
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.805652-06:00
fest_updated: 2026-06-15T16:23:52.120289-06:00
fest_tracking: true
---


# Task: Miscellaneous Small Fixes

## Objective

Apply the R36 batch of small correctness and polish fixes plus the UX-16/17/18/22 error-message style fixes: `camp commit --amend` auto-stage guard, index.lock minimum age floor, marker-before-symlink ordering, broken-symlink warning, dungeon hint correction, triage warnings to stderr, pathsafe merged into pathutil, shared slug package extracted, ID-scheme matrix documented, and the four error-message style batches.

Findings: R36 (GIT-6, GIT-9, LINK-1, LINK-2, dungeon hint, ARCH-8), cross-cutting ID/slug observations, UX-16, UX-17, UX-18, UX-22.

## Requirements

- [x] **GIT-6: `--amend` no auto-stage**: `cmd/camp/commit.go:56` defaults `-a`/`--all` to `true`. Under `--amend`, auto-staging folds unrelated WIP into the amended commit. When `--amend` is set and `-a` was not explicitly provided, default `--all` to `false`. Add `--no-edit` flag for message-less amends to avoid headless editor hang in `internal/git/commit.go:47-68`.
- [x] **GIT-9: index.lock minimum age floor**: `internal/git/lock_unix.go` decides staleness purely via open-fd check (fuser/lsof). Add a minimum lock age (e.g., 5 seconds) so a lock that was created very recently (process between lock creation and fd open) is never classified stale regardless of fd state.
- [x] **LINK-1: Marker-before-symlink ordering**: `internal/project/link.go:126-140` creates the symlink first (line 132 area), then writes the `.camp` marker (line 143 area). A crash between these two leaves a symlink whose target has no marker. Swap the order: write marker first, remove marker on symlink failure.
- [x] **LINK-2: Broken-symlink warning**: `internal/project/list.go:41-51` silently `continue`s when `filepath.EvalSymlinks` fails on a project entry (broken symlink). Emit a warning to stderr instead of silently skipping.
- [x] **Dungeon crawl hint**: `cmd/camp/dungeon/crawl.go` Long help (line 36 area) says "camp dungeon init"; the command is `camp dungeon add`. Fix all occurrences in that file.
- [x] **Triage warnings to stderr**: `internal/dungeon/triage.go` prints `fmt.Printf("Error: ...")` and `fmt.Printf("Warning: ...")` (lines 95, 104, 112 area) to stdout. Move these to stderr (`fmt.Fprintf(os.Stderr, ...)`).
- [x] **ARCH-8: Merge `pathsafe` into `pathutil`**: `internal/pathsafe` is a single file (`segment.go`) with 3 importers. Move `ValidateSegment` and `SegmentPattern` into `internal/pathutil`, delete `internal/pathsafe`, update all 3 import paths.
- [x] **Shared slug package**: `internal/intent/slug.go:30-65` and `internal/quest/slug.go:21-42` implement identical `GenerateSlug` algorithms (same regex vars, same 5-word/50-char limits, different package-level names). Extract into `internal/slug` package. Both `intent` and `quest` import and delegate to it. No behavior change.
- [x] **ID-scheme matrix doc**: document the three ID formats and their collision strategies in `docs/id-schemes.md`. The matrix is a cross-cutting reference, not a code change.
- [x] **UX-16 backwards double-wrap**: three sites use `camperrors.Wrapf(fmt.Errorf("..."), "hint %s", arg)` which renders the cause before the hint. Replace with `camperrors.Newf("hint %s: inner msg", arg)`:
  - `cmd/camp/pins.go:199`
  - `cmd/camp/leverage/dir.go:42`
  - `cmd/camp/leverage/dir.go:162`
- [x] **UX-17 multiline padded errors to single-line parenthetical hints**: seven sites use `\n       Usage: ...` or `\n       Use ...` with hardcoded seven-space padding. Replace with `(use 'camp X' to Y)` parenthetical hints:
  - `cmd/camp/switch.go:87`
  - `cmd/camp/init/command.go:166`
  - `cmd/camp/init/command.go:239`
  - `cmd/camp/quest/create.go:74-77`
  - `internal/commands/workitem/workitem.go:63`
  - `internal/commands/flow/add.go:163-168`
  - `cmd/camp/intent/add.go:163`
- [x] **UX-18 hint consistency and bare exit status**: `cmd/camp/status.go:75` returns `gitCmd.Run()` directly, which surfaces as `Error: exit status 1` with no context. Wrap the error to include the target path. Also add the `camp project list` hint to errors that hint at listing commands but currently do not.
- [x] **UX-22 stream-aware ui helpers**: `cmd/camp/pull.go`, `cmd/camp/push.go`, `cmd/camp/doctor.go`, and `cmd/camp/clone.go` use `fmt.Println(ui.Error(...))` and `fmt.Println(ui.Warning(...))` patterns that write diagnostics to stdout. Create a helper (or use `cmd.ErrOrStderr()`) to route error and warning output to stderr. Lint for `fmt.Println(ui.Error` occurrences.

## Implementation

### Background

All file:line numbers verified against the worktree at `projects/worktrees/camp/camp-hardening`. Re-verify before editing; lines may have drifted from prior sequence changes.

Work each item independently. Run the both-profile gate after each item before moving to the next.

### Step 1: GIT-6 -- amend auto-stage guard and --no-edit

**File: `cmd/camp/commit.go`**

The `--all` flag defaults to `true` (line 56 area):
```go
commitCmd.Flags().BoolVarP(&commitAll, "all", "a", true, "Stage all changes before committing")
commitCmd.Flags().BoolVar(&commitAmend, "amend", false, "Amend the previous commit")
```

After parsing flags, in `runCommit` (or wherever the amend path diverges), check whether `--all` was explicitly set by the user:

```go
// If amending and --all was not explicitly provided, do not auto-stage.
allExplicit := cmd.Flags().Changed("all")
if commitAmend && !allExplicit {
    commitAll = false
}
```

`cmd.Flags().Changed("all")` returns true only if the user explicitly passed `-a` or `--all`.

Add `--no-edit` flag:
```go
commitCmd.Flags().BoolVar(&commitNoEdit, "no-edit", false, "Amend without editing the commit message (requires --amend)")
```

In `internal/git/commit.go`, `executeCommit` builds git args. Add:
```go
if opts.NoEdit {
    args = append(args, "--no-edit")
}
```

Add `NoEdit bool` to `CommitOptions` in `internal/git/commit.go`.

**Verify the headless editor hang**: when `--amend` is passed with no `-m` and no `--no-edit`, `executeCommit` currently builds `git commit --amend` with no `-m`, which causes git to open `$EDITOR`. With `cmd.CombinedOutput()` (no stdin/stdout/stderr attached from a terminal), git either hangs waiting for input or errors immediately. Test: run `camp commit --amend` with no `-m` in a container and confirm it now errors cleanly or uses `--no-edit` as the default for the no-message-amend path.

Add a test in `internal/git/commit_test.go` asserting that `CommitOptions{Amend: true, NoEdit: true}` produces `git commit --amend --no-edit` args.

### Step 2: GIT-9 index.lock minimum age floor

**File: `internal/git/lock_unix.go`**

`IsLockStale` delegates to `checkWithFuser` or `checkWithLsof`, both of which return `stale=true` when no process has the file open. A newly created lock whose process has not yet opened the fd will be reported stale.

Add a minimum age check after the fd check:

```go
const minLockAge = 5 * time.Second

// IsLockStale checks if an index.lock file is from a dead process.
func IsLockStale(ctx context.Context, lockPath string) (bool, *LockInfo, error) {
    info := &LockInfo{Path: lockPath}

    // Age floor: never classify a very recent lock as stale.
    // A live git process between lock creation and fd open would appear stale.
    if stat, err := os.Stat(lockPath); err == nil {
        if time.Since(stat.ModTime()) < minLockAge {
            info.Stale = false
            return false, info, nil
        }
    }

    // ... existing fuser/lsof checks ...
```

The `minLockAge` constant should be exported or at least named so tests can override it. Alternatively, accept it as a parameter and call `IsLockStaleWithAge(ctx, lockPath, minLockAge)` internally.

Add a unit test in `internal/git/lock_unix.go`'s test file (or a new `lock_age_test.go`):

```go
func TestIsLockStale_RecentLockNotStale(t *testing.T) {
    tmpDir := initTestRepo(t)
    lockPath := filepath.Join(tmpDir, ".git", "index.lock")
    if err := os.WriteFile(lockPath, nil, 0644); err != nil {
        t.Fatal(err)
    }
    // File is brand-new; should not be classified stale regardless of fd state.
    stale, _, err := IsLockStale(context.Background(), lockPath)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if stale {
        t.Error("a just-created lock should not be classified stale")
    }
}
```

### Step 3: LINK-1 marker-before-symlink ordering

**File: `internal/project/link.go`**

Current order (lines 126-143 area):
1. `os.MkdirAll(filepath.Dir(fullPath), 0755)` -- create parent
2. `os.Symlink(absLocal, fullPath)` -- create symlink
3. On symlink error: return
4. `campaign.WriteMarker(absLocal, marker)` -- write .camp marker
5. On marker error: `os.Remove(fullPath)` -- remove symlink

Swap to:
1. `os.MkdirAll(filepath.Dir(fullPath), 0755)` -- create parent
2. `campaign.WriteMarker(absLocal, marker)` -- write .camp marker FIRST
3. On marker error: return
4. `os.Symlink(absLocal, fullPath)` -- create symlink
5. On symlink error: `campaign.RemoveMarker(absLocal)` (or `os.Remove(markerPath)`) -- remove marker

Check whether `campaign.RemoveMarker` or an equivalent removal function exists; if not, use:
```go
os.Remove(filepath.Join(absLocal, campaign.LinkMarkerFile))
```

The resulting code guarantees: if the process crashes after writing the marker but before creating the symlink, the marker exists in the target but no broken symlink exists in `projects/`. A subsequent re-link attempt will find the marker already present (handle idempotently).

### Step 4: LINK-2 broken-symlink warning

**File: `internal/project/list.go`**

Around line 47, when `filepath.EvalSymlinks` fails:
```go
if entry.Type()&os.ModeSymlink != 0 {
    resolvedPath, err := filepath.EvalSymlinks(projectPath)
    if err != nil {
        continue   // <-- silent skip
    }
```

Replace `continue` with a warning:
```go
    if err != nil {
        fmt.Fprintf(os.Stderr, "warning: project symlink %q is broken: %v\n", projectPath, err)
        continue
    }
```

The warning goes to stderr so it does not corrupt any JSON or table output on stdout.

### Step 5: Dungeon crawl hint

**File: `cmd/camp/dungeon/crawl.go`**

Search the file for `dungeon init` (there may be occurrences in the Long help text and the command registration). Replace each `dungeon init` with `dungeon add`:

```bash
grep -n "dungeon init" cmd/camp/dungeon/crawl.go
```

Fix each occurrence. The command `camp dungeon init` does not exist; `camp dungeon add` is the correct command.

### Step 6: Triage warnings to stderr

**File: `internal/dungeon/triage.go`**

Find `fmt.Printf("Error: ...)` and `fmt.Printf("Warning: ...)` (lines 95, 104, 112 area). Replace each with `fmt.Fprintf(os.Stderr, ...)`:

```go
// Before:
fmt.Printf("Error: %v\n", err)
// After:
fmt.Fprintf(os.Stderr, "Error: %v\n", err)
```

Import `"os"` if not already imported. Do not change non-warning/error output that legitimately belongs on stdout.

### Step 7: ARCH-8 merge pathsafe into pathutil

**Verified importers of `internal/pathsafe` (all three are non-test):**
- `internal/commands/workflow/create.go`
- `internal/commands/workitem/create.go`
- `internal/workitem/links/validate.go`

**Steps:**

1. Copy `ValidateSegment` and `SegmentPattern` from `internal/pathsafe/segment.go` into `internal/pathutil/segment.go` (new file in the pathutil package). Update the package declaration to `package pathutil`.

2. Add a one-line doc comment on the `pathutil` package (if none exists) referencing the split: `// Package pathutil provides campaign path containment, boundary validation, and path segment safety checks.`

3. In each of the three importers, replace:
   ```go
   "github.com/Obedience-Corp/camp/internal/pathsafe"
   ```
   with:
   ```go
   "github.com/Obedience-Corp/camp/internal/pathutil"
   ```
   And rename call sites from `pathsafe.ValidateSegment(...)` to `pathutil.ValidateSegment(...)`.

4. Delete `internal/pathsafe/` directory entirely.

5. Run `go build ./...` to verify no import remains.

### Step 8: Shared slug package

**New file: `internal/slug/slug.go`**

```go
// Package slug generates filesystem-safe slugs from human text.
package slug

import (
    "regexp"
    "strings"
)

var (
    whitespacePattern = regexp.MustCompile(`\s+`)
    nonAlphanumeric   = regexp.MustCompile(`[^a-z0-9-]+`)
    multipleHyphens   = regexp.MustCompile(`-+`)
)

// Generate creates a URL-safe slug from a title.
// The slug is lowercased, non-alphanumeric characters replaced, limited to
// 5 words and 50 characters.
func Generate(title string) string {
    s := strings.ToLower(title)
    s = whitespacePattern.ReplaceAllString(s, "-")
    s = nonAlphanumeric.ReplaceAllString(s, "")
    s = multipleHyphens.ReplaceAllString(s, "-")
    s = strings.Trim(s, "-")
    if s == "" {
        return ""
    }
    words := strings.Split(s, "-")
    if len(words) > 5 {
        words = words[:5]
    }
    s = strings.Join(words, "-")
    if len(s) > 50 {
        s = strings.TrimRight(s[:50], "-")
    }
    return s
}
```

**Update `internal/intent/slug.go`**: replace the local `GenerateSlug` implementation body with a delegation to `slug.Generate`:

```go
import "github.com/Obedience-Corp/camp/internal/slug"

func GenerateSlug(title string) string {
    return slug.Generate(title)
}
```

Remove the three package-level regex vars in `intent/slug.go` (they are now in `internal/slug`).

**Update `internal/quest/slug.go`**: same delegation:

```go
import "github.com/Obedience-Corp/camp/internal/slug"

func GenerateSlug(name string) string {
    return slug.Generate(name)
}
```

Remove `questWhitespacePattern`, `questNonAlphanumeric`, `questMultipleHyphens` from `quest/slug.go`.

**Add a test**: `internal/slug/slug_test.go` with table-driven cases covering empty input, multi-word, unicode, length truncation, and hyphen trimming. Verify the intent and quest slug tests still pass (they delegate, so behavior must not change).

**Import cycle check**: `internal/slug` imports only `regexp` and `strings` (stdlib). Neither `internal/intent` nor `internal/quest` is imported by `internal/slug`. No cycle.

### Step 9: ID-scheme matrix doc

**New file: `docs/id-schemes.md`**

Content (write this file):

```markdown
# ID Schemes in camp

Three distinct ID formats exist. New code must use the workitem pattern.

## Intent IDs

Format: `<slug>-YYYYMMDD-HHMMSS`
Example: `add-dark-mode-toggle-20260119-153412`
Source: `internal/intent/slug.go:GenerateID`
Collision strategy: seconds-resolution timestamp; if two intents are
captured in the same second, the second create returns an error
(collide-then-error). The caller must retry.
Location on disk: `workflow/intents/<status>/<id>/` or `intents/<status>/<id>/`

## Quest IDs

Format: `qst_YYYYMMDD_<6-char-random-hex>`
Example: `qst_20260119_a3f9c2`
Source: `internal/quest/slug.go:GenerateDirectorySlug`
Collision strategy: crypto-random 6-char suffix; uniqueness is never
verified at create time (known gap, tracked).
Location on disk: `.campaign/quests/<id>/`

## Workitem IDs

Format: `<type>-<slug>-YYYY-MM-DD`
Example: `feature-add-dark-mode-2026-01-19`
Source: `internal/commands/workitem/create.go:169-208`
Collision strategy: full collision scan at create time; if a collision is
found, a random 6-char hex suffix is appended and the scan retries. This
is the most robust strategy and must be used for any new ID-generating code.
Location on disk: `workflow/<type>/<id>/`
WI-ref format: `WI-<sha256-of-id/6-chars>` (deterministic, re-rolls on
collision, source: `internal/workitem/ref.go`).

## Convergence policy

New code generating IDs for persistent filesystem items must follow the
workitem pattern: generate a slug-based name, scan for collisions, append
a random hex suffix and retry on collision. The intent seconds-collision
error and the quest unverified-unique patterns are known gaps documented
here for reference, not as patterns to copy.
```

### Step 10: UX-16 backwards double-wrap

Three sites use `camperrors.Wrapf(fmt.Errorf("inner"), "hint %s", arg)`. When rendered, this produces `hint arg: inner` -- the inner synthetic error appears as the cause, which is backwards since there is no real underlying error.

**`cmd/camp/pins.go:199`**:
```go
// Before:
return pins.Pin{}, "", camperrors.Wrapf(fmt.Errorf("outside campaign root"),
    "pin path %q (run 'camp attach %s' first to bind this directory to the campaign)",
    absPath, absPath)

// After:
return pins.Pin{}, "", camperrors.Newf(
    "pin path %q is outside the campaign root (run 'camp attach %s' first to bind this directory to the campaign)",
    absPath, absPath)
```

**`cmd/camp/leverage/dir.go:42`**:
```go
// Before:
return nil, camperrors.Wrapf(fmt.Errorf("path is not a directory"), "%s", targetDir)

// After:
return nil, camperrors.Newf("%s: path is not a directory", targetDir)
```

**`cmd/camp/leverage/dir.go:162`** (similar pattern with "no commits for author"):
```go
// Before:
return camperrors.Wrapf(fmt.Errorf("no commits for author"), "%s in %s", authorFilter, proj.Name)

// After:
return camperrors.Newf("no commits for author %q in %s", authorFilter, proj.Name)
```

### Step 11: UX-17 multiline padded errors

Replace each `\n       Usage: ...` or similar pattern with a single-line parenthetical. The seven-space padding was designed to align with the `Error: ` prefix from `main.go:19`; it does not belong in error messages.

**`cmd/camp/switch.go:87`**:
```go
// Before:
return fmt.Errorf("campaign name required in non-interactive mode\n       Usage: camp switch <name> --print")

// After:
return camperrors.New("campaign name required in non-interactive mode (use 'camp switch <name>' or run interactively)")
```

**`cmd/camp/init/command.go:166`**:
```go
// Before:
return camperrors.New(fmt.Sprintf("already inside campaign '%s' at %s\n       Use 'camp init --repair' to add missing files", name, existingRoot))

// After:
return camperrors.Newf("already inside campaign %q at %s (use 'camp init --repair' to add missing files)", name, existingRoot)
```

**`cmd/camp/init/command.go:239`**:
```go
// Before:
return camperrors.New("repair requires confirmation\n       Use --yes to skip the prompt in non-interactive mode")

// After:
return camperrors.New("repair requires confirmation (use --yes to skip the prompt in non-interactive mode)")
```

**`cmd/camp/quest/create.go:74-77`**:
```go
// Before:
return camperrors.New("quest name is required in non-interactive mode\n       Provide a name argument or use the interactive TUI")

// After:
return camperrors.New("quest name is required in non-interactive mode (provide a name argument or run interactively)")
```

**`internal/commands/workitem/workitem.go:63`**:
```go
// Before:
return fmt.Errorf("non-interactive use requires --json or --print flag")

// After (use camperrors.New and add the hint):
return camperrors.New("non-interactive use requires --json or --print flag")
```

**`internal/commands/flow/add.go:163-168`** (three branches):
```go
// Before:
return nil, fmt.Errorf("--name is required in non-interactive mode\n       Use -n/--name flag, or run in an interactive terminal")
return nil, fmt.Errorf("--description is required in non-interactive mode\n       Use -d/--description flag, or run in an interactive terminal")
return nil, fmt.Errorf("--name and --description are required in non-interactive mode\n       Use -n/--name and -d/--description flags, --json, or run in an interactive terminal")

// After:
return nil, camperrors.New("--name is required in non-interactive mode (use -n/--name flag or run interactively)")
return nil, camperrors.New("--description is required in non-interactive mode (use -d/--description flag or run interactively)")
return nil, camperrors.New("--name and --description are required in non-interactive mode (use -n/--name, -d/--description, or run interactively)")
```

**`cmd/camp/intent/add.go:163`**:
```go
// Before:
return camperrors.Wrap(camperrors.ErrInvalidInput, "title argument required in non-interactive mode\n       Usage: camp intent add <title> [flags]")

// After:
return camperrors.Wrap(camperrors.ErrInvalidInput, "title argument required in non-interactive mode (use 'camp intent add <title>')")
```

### Step 12: UX-18 bare exit status in status.go

**File: `cmd/camp/status.go`**

Line ~75: `return gitCmd.Run()` surfaces as `Error: exit status 1` with no context of which repo or what failed.

Wrap the error:
```go
if err := gitCmd.Run(); err != nil {
    return camperrors.Wrapf(err, "git status failed for %s", target.Path)
}
return nil
```

This matches the approach in `pull.go:119-123` which classifies into `CommandError`. Here a simple wrap is sufficient since git's stderr is already passed through to the user's terminal.

### Step 13: UX-22 stream-aware ui diagnostics

The issue is that `fmt.Println(ui.Error(...))` and `fmt.Println(ui.Warning(...))` write diagnostic output to stdout. Human-mode diagnostics should go to stderr.

For each affected site, change to write to stderr. The cleanest approach is to use `fmt.Fprintln(os.Stderr, ui.Error(...))` or `fmt.Fprintln(cmd.ErrOrStderr(), ui.Error(...))` where a `*cobra.Command` is in scope.

Key sites:

**`cmd/camp/pull.go`**: per-repo error lines printed with `fmt.Println(red.Render(e))` in the summary block (around line 378). These are failure annotations and belong on stderr. Use `fmt.Fprintln(os.Stderr, red.Render(e))`.

**`cmd/camp/push.go:216`**: `fmt.Println(red.Render(e))` in the failure list. Same fix.

**`cmd/camp/doctor.go:199`**: `fmt.Println(ui.Error(...))` in fix output. If this is part of the human-readable report (always on stdout when text mode), leave it. Only move lines that are error annotations, not report body.

**`cmd/camp/clone.go`**: `fmt.Printf("  %s %s - %v\n", ui.ErrorIcon(), ...)` per-submodule failure output (line ~222). Route through stderr.

The `fmt.Println(ui.Error` lint pattern: after fixing the sites, add a comment in the `internal/ui/text.go` package doc noting that `ui.Error`/`ui.Warning` return strings intended for stderr and callers should not route them through `fmt.Println`. A full lint rule is out of scope.

### Out of scope

- Standardizing all error packages to `camperrors` CLI-wide (the `fmt.Errorf` sweep in R24 / ERR-1 is a separate sequence).
- Extracting the full shared status-dir move primitive (R25, separate large task).
- Converting `workitem.go:63` from `fmt.Errorf` to `camperrors.New` beyond what is done in Step 11 (the broader `fmt.Errorf` lint ratchet is R24).

## Completion Notes

- Implemented the `camp commit --amend` guard so implicit auto-stage is disabled for amend commits unless `--all` is explicit, added `--no-edit`, rejected invalid message-less amend forms, and added command and git-arg tests plus manual amend smoke checks.
- Added a 5-second recent-lock floor to stale `index.lock` detection while preserving active-lock PID reporting, then aged stale-lock test fixtures so stale-path coverage remains valid.
- Reordered project linking to write `.camp` markers before symlinks and roll markers back on symlink failure; broken project symlinks now warn to stderr.
- Corrected dungeon stale hints from `camp dungeon init` to `camp dungeon add`; the stale hint was in `internal/dungeon/crawl.go`, not `cmd/camp/dungeon/crawl.go`.
- Moved dungeon triage diagnostics and pull/push/clone/doctor error annotations to stderr where they are diagnostics, and documented the stderr expectation on the UI helpers.
- Merged `internal/pathsafe` into `internal/pathutil`, extracted shared slug generation into `internal/slug`, and added `docs/id-schemes.md`.
- Replaced UX-16 double-wraps with direct `camperrors.New` messages without `fmt.Errorf` inner causes. The current camperrors package does not provide `Newf`, so formatted messages use `camperrors.New(fmt.Sprintf(...))`.
- Replaced the known padded multiline errors with single-line parenthetical hints and added the `camp project list` hints/status path context requested by UX-18.

## Done When

- [x] Both-profile gate green: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`
- [x] `camp commit --amend -m "fix"` does not auto-stage unrelated tracked changes (verified in integration or by code inspection)
- [x] `camp commit --amend --no-edit` succeeds without editor hang
- [x] `internal/git/lock_unix.go` applies minimum age floor; `TestIsLockStale_RecentLockNotStale` passes
- [x] `internal/project/link.go` writes marker before symlink (verified by code inspection of the ordering)
- [x] `camp project list` in a campaign with a broken symlink prints a warning to stderr and continues
- [x] `camp dungeon crawl --help` says `dungeon add` not `dungeon init`
- [x] `internal/dungeon/triage.go` error/warning output goes to stderr (`grep -n "fmt.Printf.*Error\|fmt.Printf.*Warning" internal/dungeon/triage.go` returns zero hits)
- [x] `internal/pathsafe` directory does not exist; all 3 former importers use `pathutil.ValidateSegment`
- [x] `internal/slug/slug.go` exists; `internal/intent` and `internal/quest` slug tests still pass
- [x] `docs/id-schemes.md` exists with the three ID formats documented
- [x] `cmd/camp/pins.go:199`, `leverage/dir.go:42`, `leverage/dir.go:162` use direct `camperrors.New` messages with no `fmt.Errorf` inner
- [x] All seven UX-17 multiline padded errors are single-line parenthetical hints
- [x] `camp status` on a failed git repo returns an error message containing the target path
- [x] No `fmt.Println(ui.Error` or `fmt.Println(red.Render` diagnostic patterns in pull.go, push.go per-failure lists