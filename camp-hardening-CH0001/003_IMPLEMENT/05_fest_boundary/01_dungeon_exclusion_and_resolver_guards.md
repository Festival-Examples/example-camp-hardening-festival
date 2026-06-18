---
fest_type: task
fest_id: 01_dungeon_exclusion_and_resolver_guards.md
fest_name: dungeon_exclusion_and_resolver_guards
fest_parent: 05_fest_boundary
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.675588-06:00
fest_updated: 2026-06-14T21:47:28.530521-06:00
fest_tracking: true
---


# Task: Dungeon Exclusion Map and Resolver Fest-Ownership Guards

## Objective

Add `festivals` and `projects` to the dungeon triage exclusion map, make the dungeon resolver skip any `dungeon/` directory whose parent chain contains a `festivals/` path element, and scaffold a default `dungeon/.crawl.yaml` at campaign init so that camp can never relocate or adopt fest-owned state (WF-1, WF-2 from findings-subsystems.md).

## Requirements

- [x] `festivals` and `projects` are present in the built-in exclusion map in `internal/dungeon/service_list.go:125-136`; `camp dungeon crawl` and triage at the campaign root never present either directory as a candidate.
- [x] `internal/dungeon/resolver.go:64-86` skips any candidate `<dir>/dungeon` directory when `dir`'s path contains a `festivals` element; the resolver continues walking upward instead of returning the fest-owned directory.
- [x] `internal/dungeon/scaffold/init.go` writes a default `dungeon/.crawl.yaml` that excludes `festivals` and `projects` in the `excludes` list; the write is skipped when the file already exists (idempotent, matching the OBEY.md behavior in the same file).
- [x] Regression test `TestListParentItems_ExcludesFestivals`: a campaign-root fixture containing a `festivals/` directory does not return that directory in `ListParentItems` results.
- [x] Regression test `TestListParentItems_ExcludesProjects`: same fixture with a `projects/` directory.
- [x] Regression test `TestResolveContext_IgnoresFestivalsDungeon`: `ResolveContext` with cwd set to `festivals/planning/my-fest/` inside a campaign that has both `festivals/dungeon/` and a campaign-root `dungeon/` returns the campaign-root dungeon, never `festivals/dungeon/`.
- [x] Regression test `TestResolveContext_NoDungeonInFestivalsSubtree`: `ResolveContext` with cwd inside `festivals/` when only `festivals/dungeon/` exists (no campaign-root `dungeon/`) returns `ErrDungeonContextNotFound`, never returns the fest-owned directory.
- [x] A dated-bucket write path test: confirm that a dungeon move from the campaign root does not write a bucket under `festivals/`.

## Implementation

### Background: why this matters

`camp dungeon move festivals archived` at the campaign root calls `os.Rename` on the entire `festivals/` tree, moving it to `dungeon/archived/YYYY-MM-DD/festivals`. One keystroke relocates fest's entire state directory, silently, with an auto-commit. The auto-commit makes it harder to recover because the rename appears intentional in git history.

The resolver problem is the mirror: if you run any camp dungeon command while cwd is inside `festivals/planning/<some-fest>/`, the resolver walks upward and finds `festivals/dungeon/` before reaching the campaign root. From that point camp treats `festivals/dungeon/` as the active camp dungeon context, appends its `crawl.jsonl` there, and `camp dungeon add` will scaffold `OBEY.md` and `.gitkeep` files inside fest's terminal-status directory.

fest's terminal-status directory is exactly `festivals/dungeon/`. Moving it or writing into it corrupts fest's state.

### Step 1: Add `festivals` and `projects` to the built-in exclusion map

File: `internal/dungeon/service_list.go`

Verified location at `bc2ad1f`: the `excluded` map literal spans lines 125-136. It already excludes `dungeon`, `.campaign`, `.git`, several doc files, and dot-file names. It does NOT exclude `festivals` or `projects`.

Add the two entries to the map:

```go
excluded := map[string]bool{
    "dungeon":      true,
    ".campaign":    true,
    ".git":         true,
    "AGENTS.md":    true,
    "CLAUDE.md":    true,
    "OBEY.md":      true,
    "README.md":    true,
    ".gitkeep":     true,
    ".gitignore":   true,
    ".crawlignore": true,
    "festivals":    true,
    "projects":     true,
}
```

These two entries are the minimal required fix. The `.workflow.yaml` exclusion layer (lines 138-148) and the `.crawl.yaml` exclusion layer (lines 150-161) provide additional defense in depth for users with custom setups, but they do not fire for a real campaign root by default (no `OBEY.md` exists inside `festivals/`, no root `.workflow.yaml` exists, and no default `dungeon/.crawl.yaml` is scaffolded today). The built-in map is the only reliable gate.

### Step 2: Add the resolver fest-ownership check

File: `internal/dungeon/resolver.go`

Verified location at `bc2ad1f`: the walk loop spans lines 64-86. The loop picks the first `<dir>/dungeon` that exists as a directory. There is no check for fest ownership.

The fix: inside the `case statErr == nil && info.IsDir():` branch, before returning, check whether any element of `dir`'s path is `festivals`. If it is, skip this candidate and continue the walk upward.

The element check must use path element comparison (not `strings.Contains`) to avoid false matches like a directory named `my-festivals-data`. The pattern is:

```go
// isFestOwned returns true when the directory path contains a path element
// named "festivals", meaning this dungeon/ is inside fest's state tree.
func isFestOwned(dir string) bool {
    for _, elem := range splitPathElements(dir) {
        if elem == "festivals" {
            return true
        }
    }
    return false
}

// splitPathElements splits a cleaned absolute path into its individual
// directory name components, skipping empty strings from leading slashes.
func splitPathElements(p string) []string {
    clean := filepath.Clean(p)
    parts := strings.Split(clean, string(filepath.Separator))
    var elems []string
    for _, part := range parts {
        if part != "" {
            elems = append(elems, part)
        }
    }
    return elems
}
```

Then modify the walk loop in `ResolveContext`:

```go
for {
    candidate := filepath.Join(dir, "dungeon")
    info, statErr := os.Stat(candidate)
    switch {
    case statErr == nil && info.IsDir():
        if isFestOwned(dir) {
            // This dungeon/ lives inside fest's state tree; skip it.
            break
        }
        return Context{
            DungeonPath: candidate,
            ParentPath:  dir,
        }, nil
    case statErr != nil && !errors.Is(statErr, os.ErrNotExist):
        return Context{}, camperrors.Wrapf(statErr, "stat %s", candidate)
    }
    // ... rest of loop unchanged
```

The `strings` import is already available in the package (check the import block); if not, add it. `filepath` is already imported. Note that by the time the loop runs, `dir` has already been resolved through `filepath.EvalSymlinks` at lines 55-62. On macOS, `/var` resolves to `/private/var`, so paths like `/private/var/folders/.../festivals/planning/...` are already canonical before the element comparison runs. The element check on the canonical path is safe.

The `isFestOwned` and `splitPathElements` helpers should be package-private functions in a new small file (e.g., `internal/dungeon/boundary.go`) or added to `resolver.go`. Keep them unexported; they are resolver implementation details.

Edge cases:
- cwd is campaign root itself: `dir` is the campaign root, which should not contain `festivals` as an element (unless someone named their campaign root `festivals`, which is unusual and not a supported case). The campaign root's dungeon is returned normally.
- cwd is inside `festivals/dungeon/planning/` (someone nested deeper): every upward step from within the `festivals/` subtree has `festivals` as a path element, so all candidates are skipped until the walk reaches the campaign root (where `dir` is the campaign root path, not containing `festivals`).
- No campaign-root dungeon exists: the loop reaches `dir == root` without finding an acceptable candidate and returns `ErrDungeonContextNotFound`. This is correct; the user should run `camp dungeon add` from the campaign root first.

### Step 3: Scaffold a default `dungeon/.crawl.yaml` at campaign init

File: `internal/dungeon/scaffold/init.go`

The `Init` function already creates the dungeon directory, OBEY.md, and `.gitkeep` files for each status directory. Add a `.crawl.yaml` creation step with the same skip-if-exists semantics used for OBEY.md (lines 57-73).

Add the following after the existing `.gitkeep` loop (after line 87, before the final `return result, nil`):

```go
crawlCfgPath := filepath.Join(dungeonPath, ".crawl.yaml")
if err := ctx.Err(); err != nil {
    return nil, camperrors.Wrap(err, "context cancelled")
}

if _, err := os.Stat(crawlCfgPath); os.IsNotExist(err) {
    const defaultCrawlConfig = "excludes:\n  - festivals\n  - projects\n"
    if err := os.WriteFile(crawlCfgPath, []byte(defaultCrawlConfig), 0644); err != nil {
        return nil, camperrors.Wrapf(err, "writing %s", crawlCfgPath)
    }
    result.CreatedFiles = append(result.CreatedFiles, crawlCfgPath)
} else if err == nil && opts.Force {
    const defaultCrawlConfig = "excludes:\n  - festivals\n  - projects\n"
    if err := os.WriteFile(crawlCfgPath, []byte(defaultCrawlConfig), 0644); err != nil {
        return nil, camperrors.Wrapf(err, "writing %s", crawlCfgPath)
    }
} else if err == nil {
    result.Skipped = append(result.Skipped, crawlCfgPath)
}
```

This is defense in depth. If a user removes the hardcoded `festivals` entry from the `excluded` map (which they cannot via config), the `.crawl.yaml` still prevents `festivals` from surfacing. More practically, the `.crawl.yaml` defense fires for users who have already-initialized dungeons once the binary is updated, without needing to re-run `camp dungeon add`.

The `.crawl.yaml` format is defined in `internal/dungeon/crawl_config.go`: `CrawlConfig` has an `Excludes []string` field marshaled as `yaml:"excludes"`. The YAML content above is exactly what `yaml.Marshal` would produce for `CrawlConfig{Excludes: []string{"festivals", "projects"}}`.

### Step 4: Write regression tests

File: `internal/dungeon/service_list_test.go` (existing, add new test functions)

Follow the established pattern in `TestListParentItems_ExcludesOBEYManagedDirs` (line 218 in the worktree). Use `t.TempDir()` for fixtures. The host t.TempDir convention is correct here per C-9 (these are guard checks, not destructive filesystem mutation tests requiring containerization).

```go
func TestListParentItems_ExcludesFestivals(t *testing.T) {
    root := t.TempDir()
    // Create a festivals/ directory with content that looks like real fest state.
    if err := os.MkdirAll(filepath.Join(root, "festivals", "planning"), 0755); err != nil {
        t.Fatal(err)
    }
    if err := os.MkdirAll(filepath.Join(root, "festivals", "dungeon"), 0755); err != nil {
        t.Fatal(err)
    }
    // Create a legitimate triage candidate alongside it.
    if err := os.MkdirAll(filepath.Join(root, "my-work"), 0755); err != nil {
        t.Fatal(err)
    }
    dungeonPath := filepath.Join(root, "dungeon")
    svc := NewService(root, dungeonPath)
    items, err := svc.ListParentItems(context.Background(), root)
    if err != nil {
        t.Fatal(err)
    }
    for _, item := range items {
        if item.Name == "festivals" {
            t.Errorf("festivals should be excluded from triage candidates, but got item: %+v", item)
        }
    }
}

func TestListParentItems_ExcludesProjects(t *testing.T) {
    root := t.TempDir()
    if err := os.MkdirAll(filepath.Join(root, "projects", "camp"), 0755); err != nil {
        t.Fatal(err)
    }
    if err := os.MkdirAll(filepath.Join(root, "my-work"), 0755); err != nil {
        t.Fatal(err)
    }
    dungeonPath := filepath.Join(root, "dungeon")
    svc := NewService(root, dungeonPath)
    items, err := svc.ListParentItems(context.Background(), root)
    if err != nil {
        t.Fatal(err)
    }
    for _, item := range items {
        if item.Name == "projects" {
            t.Errorf("projects should be excluded from triage candidates, but got item: %+v", item)
        }
    }
}
```

File: `internal/dungeon/resolver_test.go` (existing, add new test functions)

Follow the pattern of `TestResolveContext_NearestDungeon` (line 22). The test fixture must include a real `festivals/dungeon/` directory to confirm the resolver skips it.

```go
func TestResolveContext_IgnoresFestivalsDungeon(t *testing.T) {
    root := t.TempDir()
    // Set up campaign-root dungeon.
    campaignDungeon := filepath.Join(root, "dungeon")
    if err := os.MkdirAll(campaignDungeon, 0755); err != nil {
        t.Fatal(err)
    }
    // Set up fest-owned dungeon.
    festsDungeon := filepath.Join(root, "festivals", "dungeon")
    if err := os.MkdirAll(festsDungeon, 0755); err != nil {
        t.Fatal(err)
    }
    // cwd inside a festival.
    cwd := filepath.Join(root, "festivals", "planning", "my-fest")
    if err := os.MkdirAll(cwd, 0755); err != nil {
        t.Fatal(err)
    }

    ctx, err := ResolveContext(context.Background(), root, cwd)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    // Must resolve to the campaign-root dungeon, not festivals/dungeon/.
    wantDungeon, _ := filepath.EvalSymlinks(campaignDungeon)
    gotDungeon, _ := filepath.EvalSymlinks(ctx.DungeonPath)
    if gotDungeon != wantDungeon {
        t.Errorf("got DungeonPath %q, want %q", gotDungeon, wantDungeon)
    }
}

func TestResolveContext_NoDungeonInFestivalsSubtree(t *testing.T) {
    root := t.TempDir()
    // Only festivals/dungeon/ exists; no campaign-root dungeon.
    festsDungeon := filepath.Join(root, "festivals", "dungeon")
    if err := os.MkdirAll(festsDungeon, 0755); err != nil {
        t.Fatal(err)
    }
    cwd := filepath.Join(root, "festivals", "planning", "my-fest")
    if err := os.MkdirAll(cwd, 0755); err != nil {
        t.Fatal(err)
    }

    _, err := ResolveContext(context.Background(), root, cwd)
    if err == nil {
        t.Fatal("expected error when only festivals/dungeon/ exists, got nil")
    }
    if !errors.Is(err, ErrDungeonContextNotFound) {
        t.Errorf("got error %v, want ErrDungeonContextNotFound", err)
    }
}
```

Note: both resolver tests use `filepath.EvalSymlinks` when comparing paths, per the macOS `/private/var` convention documented in the campaign memory and already used in the existing resolver_test.go. Check how the existing tests do the comparison and mirror the pattern.

## Out of Scope

- Fest's own behavior is out of scope per constraint C-5. Do not modify any file in the fest project.
- The shared statusdir move primitive (sequence 11.05, R25) would eventually let `MoveToDungeon` refuse to move `festivals/` at the move layer too. That is a later sequence; this task only adds exclusion-map and resolver guards.
- Dungeon TOCTOU and stat-conflation fixes (sequence 11.05) are not part of this task.
- The `.crawl.yaml` content format itself is already defined in `internal/dungeon/crawl_config.go`; do not change the schema.

## Done When

- [x] Both `festivals` and `projects` appear in the `excluded` map at `internal/dungeon/service_list.go` (within the literal starting at line 125 in the `bc2ad1f` worktree; re-verify line by reading the file before editing).
- [x] `isFestOwned` check is present in `internal/dungeon/resolver.go` (or a separate `boundary.go` in the same package); the walk loop in `ResolveContext` skips fest-owned candidates.
- [x] `internal/dungeon/scaffold/init.go` creates `dungeon/.crawl.yaml` with `excludes: [festivals, projects]` when the file does not already exist; skips when it does.
- [x] `TestListParentItems_ExcludesFestivals` and `TestListParentItems_ExcludesProjects` pass.
- [x] `TestResolveContext_IgnoresFestivalsDungeon` and `TestResolveContext_NoDungeonInFestivalsSubtree` pass.
- [x] Both-profile gate commands pass: `just build`, `BUILD_TAGS=dev just build`, `go test -count=1 -short ./internal/dungeon/...` (stable and dev tags), `just lint`, `BUILD_TAGS=dev just lint`.

## Verification

- `go test -count=1 -short ./internal/dungeon/...`
- `BUILD_TAGS=dev go test -count=1 -short ./internal/dungeon/...`
- `just build stable`
- `just build dev`
- `just lint`
- `BUILD_TAGS=dev just lint`
- `git diff --check`

Project commit: `b382d48` (`fix: guard dungeon from fest-owned state`).