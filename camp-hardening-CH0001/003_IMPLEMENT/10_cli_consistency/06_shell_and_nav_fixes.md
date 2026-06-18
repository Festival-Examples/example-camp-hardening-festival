---
fest_type: task
fest_id: 06_shell_and_nav_fixes.md
fest_name: shell_and_nav_fixes
fest_parent: 10_cli_consistency
fest_order: 6
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.770537-06:00
fest_updated: 2026-06-15T02:54:40.900454-06:00
fest_tracking: true
---


# Task: Shell Wrapper Transparency and Nav Existence Checks

## Objective

Fix the shell wrappers so they no longer silently suppress camp stderr, add `--help`/`--json`/`--print` passthrough to the go and switch branches (matching the workitem branch), make `camp go --print` stat the resolved path before printing and retry once on miss, reject `--list` with `--print` as conflicting options, drop the incorrect `!printOnly` guard on the disambiguation warning, and lock the nav index rebuild to prevent unserialized concurrent rebuilds.

## Requirements

- [ ] `internal/shell/templates/bash.sh.tmpl` and `zsh.sh.tmpl`: in the `go|g` branch, replace `2>/dev/null` with exit-code checks; the generic "camp: not found" message is shown only when camp produced no output at all (empty `$dest` AND camp exited non-zero). Do the same for the `switch|sw` branch which has the same pattern. Apply equivalent fixes in `fish.sh.tmpl`.
- [ ] `bash.sh.tmpl`, `zsh.sh.tmpl`, `fish.sh.tmpl`: the `go|g` and `switch|sw` branches gain a passthrough scan matching the workitem branch (`bash.sh.tmpl:46-56`). When the user passes `--help`, `-h`, `--json`, `--json=*`, `--print`, or `--print=*`, forward directly to `command camp go "$@"` / `command camp switch "$@"` without intercepting for the cd behavior.
- [ ] `cmd/camp/navigation/go.go`: reject `--list` when `--print` is also set, returning an error explaining the flags are mutually exclusive.
- [ ] `cmd/camp/navigation/go.go`: drop the `!printOnly` condition on the multi-match disambiguation warning (currently `if resolveResult.HasMultipleMatches() && !printOnly`). The warning writes to stderr; `--print` callers should see it on stderr so they know the path was ambiguous.
- [ ] `cmd/camp/navigation/go.go`: before printing the resolved path under `--print`, call `os.Stat(resolveResult.Path)`; on `IsNotExist`, call `index.GetOrBuild(ctx, campaignRoot, true)` (forced rebuild), re-resolve, and stat again; if still missing, return a clear error (`camperrors.Newf("resolved path does not exist: %s", path)`). Same fix for the pins path (lines around 533-539; verify).
- [ ] `internal/nav/index/cache.go`: wrap the stale-check/rebuild/save in `GetOrBuild` with `fsutil.AcquireFileLock(cacheFilePath+".lock")`. Load and save errors that are currently swallowed (the "in production this would be logged" comments at lines 224-227 and 247-249) are replaced with `fmt.Fprintf(os.Stderr, "camp: nav cache warning: %v\n", err)` - using stderr so they do not pollute `--print` stdout.
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

The go and switch wrappers in all three shell templates use `2>/dev/null` on the `command camp go "$@" --print` call. This hides camp's error output, including the disambiguation list that `go.go` sends to stderr and any config errors. When the resolved path is empty, the wrapper shows "camp: not found" regardless of whether camp failed with an actionable error or simply found nothing. The exit code is never checked; only string emptiness is tested.

The workitem branch already handles passthrough correctly (`bash.sh.tmpl:46-56`): it scans arguments for `--help/-h/--json/--print` and forwards to `command camp workitem "$@"` directly. The go and switch branches lack this, so `camp go p x --print` in a sourced shell is intercepted by the function, cds (or does nothing), and returns nothing to stdout - silently breaking the pattern `cd "$(camp go ... --print)"`.

The nav `--print` print-without-stat bug means `cd "$(camp go --print someproject)"` can cd into a nonexistent directory when a linked project was deleted. A single forced rebuild is enough for the common case of a recently deleted project whose mtime bump invalidates the cache.

The cache rebuild race is low-priority for correctness but matters for agent loops that issue many concurrent `camp go --print` calls (festival-app polls navigation). The `fsutil.AcquireFileLock` pattern is already used for the registry (`internal/workitem/links/save.go:72-97`); apply the same approach.

Verify all cited line numbers against the worktree at `bc2ad1f` before editing.

### Verified file:line targets (as of review citations; verify before editing)

- `internal/shell/templates/bash.sh.tmpl:24,33` - `2>/dev/null` in go branch
- `internal/shell/templates/bash.sh.tmpl:46-56` - workitem passthrough (reference pattern, do not change)
- `internal/shell/templates/zsh.sh.tmpl:25,35` - `2>/dev/null` in go branch
- `cmd/camp/navigation/go.go:256-258` - `--list` branch runs before `--print` check
- `cmd/camp/navigation/go.go:265-271` - `!printOnly` gate on disambiguation warning
- `cmd/camp/navigation/go.go:273-277` - `--print` path printing without stat
- `cmd/camp/navigation/go.go:533-539` - pins path without stat
- `internal/nav/index/cache.go:216-252` - `GetOrBuild` without lock
- `internal/nav/index/cache.go:224-227` - load error swallowed
- `internal/nav/index/cache.go:247-249` - save error swallowed

### Step-by-step approach

**Step 1: Fix `2>/dev/null` in bash.sh.tmpl.**

Read the bash template. The current go branch pattern (around line 33):

```bash
dest=$(command camp go "$@" --print 2>/dev/null)
if [ -n "$dest" ]; then
    cd "$dest" || return 1
else
    echo "camp: not found: $*" >&2
    return 1
fi
```

Replace with:

```bash
dest=$(command camp go "$@" --print)
status=$?
if [ -n "$dest" ]; then
    cd "$dest" || return 1
elif [ $status -ne 0 ]; then
    return $status
else
    echo "camp: not found: $*" >&2
    return 1
fi
```

This preserves camp's stderr output (errors and the disambiguation list are shown), checks the exit code, and only shows the generic message when camp produced no output but exited zero (i.e., genuinely found nothing). Apply the same change to the no-argument variant near line 24 (the `$# -eq 0` branch) and to the `switch|sw` branch.

**Step 2: Apply the same fix in zsh.sh.tmpl and fish.sh.tmpl.**

Read each file. The zsh template has equivalent `2>/dev/null` patterns at lines 25 and 35 (verify). Fish syntax differs but the logic is the same: capture exit status with `$status` (fish's built-in), check it before showing the generic message.

Fish equivalent:

```fish
set -l dest (command camp go $rest --print)
if test -n "$dest"
    cd $dest
else if test $status -ne 0
    return $status
else
    echo "camp: not found: $rest" >&2
    return 1
end
```

**Step 3: Add passthrough scan to go/switch branches in all three templates.**

Model the change on the workitem branch (`bash.sh.tmpl:46-56`). Insert the scan before the existing logic:

```bash
go|g)
    shift
    # Passthrough for --help, --json, --print (let camp handle them directly)
    local arg passthrough=0
    for arg in "$@"; do
        case "$arg" in
            --help|-h|--json|--json=*|--print|--print=*)
                passthrough=1
                break
                ;;
        esac
    done
    if [ "$passthrough" -eq 1 ]; then
        command camp go "$@"
        return
    fi
    # ... existing go logic ...
```

Apply the same passthrough logic to the `switch|sw` branch. This means `camp go someproject --print` called directly in the terminal (outside a script) passes through to `command camp go someproject --print`, which outputs the path to stdout without the cd side effect. Inside `cd "$(camp go ... --print)"`, the outer shell function's passthrough returns the path on stdout and the calling script gets the correct value.

For fish, use the equivalent:

```fish
for arg in $rest
    if string match -qr '^(--help|-h|--json|--print)' -- $arg
        command camp go $rest
        return
    end
end
```

**Step 4: Fix `--list` + `--print` conflict in `go.go`.**

Read `cmd/camp/navigation/go.go`. Find where `listShortcuts` and `printOnly` flags are parsed. Before the resolution call, add:

```go
if listShortcuts && printOnly {
    return camperrors.New("--list and --print are mutually exclusive")
}
```

Place this early in the RunE body, after flag parsing but before any work is done.

**Step 5: Drop the `!printOnly` gate on disambiguation warning.**

Find the current code (around line 265-271):

```go
if resolveResult.HasMultipleMatches() && !printOnly {
    fmt.Fprintln(os.Stderr, ui.Warning("Multiple matches found:"))
    ...
}
```

Change to:

```go
if resolveResult.HasMultipleMatches() {
    fmt.Fprintln(os.Stderr, ui.Warning("Multiple matches found:"))
    ...
}
```

The warning writes to stderr; `--print` consumers reading stdout are unaffected.

**Step 6: Add `os.Stat` before printing in `--print` mode.**

Read the `if printOnly { fmt.Println(resolveResult.Path) }` block (around line 273-277). Replace with:

```go
if printOnly {
    if _, err := os.Stat(resolveResult.Path); os.IsNotExist(err) {
        // Stale cache: force one rebuild and re-resolve
        idx, buildErr := index.GetOrBuild(ctx, campaignRoot, true)
        if buildErr != nil {
            return camperrors.Wrapf(buildErr, "nav index rebuild failed")
        }
        // Re-resolve with the fresh index
        resolveResult, err = idx.Resolve(...)  // use same resolve call as above; extract a helper if needed
        if err != nil {
            return err
        }
        if _, statErr := os.Stat(resolveResult.Path); os.IsNotExist(statErr) {
            return camperrors.Newf("resolved path does not exist: %s (project may have been deleted)", resolveResult.Path)
        }
    }
    fmt.Println(resolveResult.Path)
    return nil
}
```

The re-resolve logic may require refactoring the resolution call into a helper function if it is currently inline. Extract it if needed; keep the change minimal otherwise. Apply the same stat-before-print fix to the pins path around line 533-539.

**Step 7: Lock the nav cache rebuild.**

Read `internal/nav/index/cache.go:GetOrBuild`. Add a lock around the stale-check/rebuild/save section:

```go
func GetOrBuild(ctx context.Context, campaignRoot string, forceRebuild bool) (*Index, error) {
    if ctx.Err() != nil {
        return nil, ctx.Err()
    }

    if !forceRebuild {
        idx, err := Load(campaignRoot)
        if err != nil {
            fmt.Fprintf(os.Stderr, "camp: nav cache warning: failed to load: %v\n", err)
        }
        if idx != nil && !IsStale(idx, campaignRoot) {
            return idx, nil
        }
    }

    // Lock around rebuild+save so concurrent processes do not double-rebuild.
    cachePath := cacheFilePath(campaignRoot) // find the cache path function
    unlock, lockErr := fsutil.AcquireFileLock(cachePath + ".lock")
    if lockErr != nil {
        fmt.Fprintf(os.Stderr, "camp: nav cache warning: could not acquire lock: %v\n", lockErr)
        // Fall through: rebuild without lock rather than fail completely.
    } else {
        defer unlock()
        // Re-check staleness after acquiring the lock (another process may have rebuilt).
        if !forceRebuild {
            if idx, err := Load(campaignRoot); err == nil && !IsStale(idx, campaignRoot) {
                return idx, nil
            }
        }
    }

    // ... existing build logic ...

    if saveErr := Save(idx, campaignRoot); saveErr != nil {
        fmt.Fprintf(os.Stderr, "camp: nav cache warning: failed to save: %v\n", saveErr)
    }

    return idx, nil
}
```

Read the `fsutil.AcquireFileLock` signature to understand its return value (likely `func() error` or `func()`). Follow the pattern from `internal/workitem/links/save.go:72-97`.

**Step 8: Golden tests for shell templates.**

Write tests in `internal/shell/` (host `t.TempDir()` convention) that render each template and verify:
1. The rendered go wrapper does NOT contain `2>/dev/null`.
2. The rendered go wrapper DOES contain a passthrough scan for `--help`.
3. The rendered go wrapper checks exit status before showing the generic message.

Use `strings.Contains` or a simple substring check against the rendered template output. Three template files, three test functions. Store expected snippets inline in the test (not golden files) to avoid the host-git allowlist restriction.

### Edge cases

**SH-3 and NAV-5:** These are in-scope-if-touched hygiene notes (from the task description). If the changes in this task touch `internal/shell/shortcuts.go` (SH-3) or `internal/nav/index/target.go` (NAV-5), apply the hygiene fix then. If not, leave them as deferred work for sequence 12.

**Fish syntax differs:** Fish does not have `local` or `[ ... ]`. Use fish-native syntax (`set -l`, `test`, `string match`). Read the existing fish workitem passthrough in `fish.sh.tmpl` for the correct pattern.

**Nav re-resolve after forced rebuild:** The re-resolve after a forced rebuild requires calling the same resolution logic a second time. If the resolution is deeply nested inline code, extract a `resolveTarget(ctx, campaignRoot, idx, args) (ResolveResult, error)` helper before adding the stat-retry. Keep the extraction change separate (commit it first) from the stat logic (commit it second).

### Out of scope

- SH-3 (completion generation escaping) is sequence 11 or 12 debt.
- NAV-5 (sub-shortcut `..` acceptance) is a hygiene note.
- Concurrent rebuild test requires container isolation per C-9; note it in the test file with a `//go:build integration` stub if desired, but do not block this task on it.

## Done When

- [ ] All requirements met
- [ ] `camp go nonexistent-project 2>&1` shows camp's real error, not the suppressed generic message
- [ ] `cd "$(camp go someproject --print)"` in a script works (passthrough honors `--print`)
- [ ] `camp go --list --print` returns an error about mutually exclusive flags
- [ ] `camp go --print` on a project whose path was deleted returns a clear "resolved path does not exist" error
- [ ] Shell template golden tests pass confirming no `2>/dev/null` and passthrough present
- [ ] Both-profile gate passes