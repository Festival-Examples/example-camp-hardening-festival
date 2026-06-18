---
fest_type: task
fest_id: 04_root_commit_scope_and_errnochanges.md
fest_name: root_commit_scope_and_errnochanges
fest_parent: 07_commit_semantics
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.714429-06:00
fest_updated: 2026-06-14T23:39:34.987747-06:00
fest_tracking: true
---


# Task: Scope root camp commit and classify both no-changes git phrases (N-14, N-23)

## Objective

Prevent `camp commit` at the campaign root from sweeping pre-staged `projects/*` gitlinks into a content commit, and ensure "no changes added to commit" is classified as `ErrNoChanges` so the refs-only-drift case produces the friendly hint rather than a raw git error.

## Requirements

- [ ] `runCommit` in `cmd/camp/commit.go:91-204` detects pre-existing staged `projects/*` gitlinks before the final commit and refuses with a clear message unless `--include-refs` is set; the message names which paths were found and suggests `camp refs-sync` or `--include-refs`.
- [ ] `ClassifyGitError` in `internal/git/errors.go:159-181` matches "no changes added to commit" in addition to "nothing to commit" for `GitErrorNoChanges`.
- [ ] The refs-only-drift case (where `StageAllExcluding` stages nothing because the only changes are submodule pointers) surfaces the existing friendly hint from `stage.go` instead of a raw git error string.
- [ ] Both fixes are active in stable and dev profiles (no build tag gating).
- [ ] Tests: pre-staged gitlink from an aborted prior run is not swept into a content commit; refs-only drift produces the friendly hint.

## Implementation

### Background

**N-14 (root commit unscoped):** `runCommit` at `:137-153` stages with submodule exclusions via `StageAllExcluding`. However the final commit at `:189-200` is unscoped (`executor.Commit(ctx, opts)` with no `Only` or `TempIndexPath`). If a `projects/*` gitlink was already staged from a previous aborted run (raw `git add`, or a concurrent fest/camp process), it is swept into this commit under the workitem or campaign tag that belongs to the content being committed.

The simplest safe fix is a pre-commit guard: detect staged `projects/*` entries before the commit and refuse. The alternative -- scoping the final commit to exactly the paths `StageAllExcluding` staged -- is more complex because `StageAllExcluding` does not return a path list. The refusal approach is chosen here (documented tradeoff: it is explicit, visible to the user, and requires no changes to the staging pipeline).

**N-23 ("no changes added to commit"):** `git commit` returns different phrasing depending on the scenario:
- When nothing at all is staged and `git commit` is run without `--allow-empty`, git prints: `nothing to commit, working tree clean`
- When the worktree has changes but nothing is staged (e.g., only refs changes after `StageAllExcluding`), git prints: `no changes added to commit (use "git add" and/or "git commit -a")`

`ClassifyGitError` at `:165` only matches "nothing to commit". The second phrase falls through to `GitErrorUnknown`, which becomes a raw `camperrors.NewGit("commit", ...)` instead of `ErrNoChanges`. The friendly refs-excluded hint in the stage path never fires because the error is not recognized as `ErrNoChanges`.

### Verified file:line targets (worktree HEAD bc2ad1f)

- `cmd/camp/commit.go:137-153` -- staging block; `StageAllExcluding` at :150.
- `cmd/camp/commit.go:188-200` -- final commit block; `executor.Commit(ctx, opts)` at :194; `errors.Is(err, git.ErrNoChanges)` at :195.
- `internal/git/errors.go:159-181` -- `ClassifyGitError`; "nothing to commit" match at :165.
- `internal/git/errors.go:98` -- `ErrNoChanges = errors.New("nothing to commit")`.

Note: the review cited `cmd/camp/commit.go:~:137-153` and `~:189-200`; both match the worktree exactly.

### Step 1: Add "no changes added to commit" to ClassifyGitError

In `internal/git/errors.go:159-181`, extend the `GitErrorNoChanges` case:

```go
// Before:
case strings.Contains(lower, "nothing to commit"):
    return GitErrorNoChanges

// After:
case strings.Contains(lower, "nothing to commit"),
    strings.Contains(lower, "no changes added to commit"):
    return GitErrorNoChanges
```

This is a one-line addition. The second phrase is emitted by git when there are unstaged changes but nothing in the index. Both phrases should map to `ErrNoChanges` because the correct response in both cases is "tell the user there is nothing to commit from the index perspective."

### Step 2: Add a pre-commit gitlink guard in runCommit

In `runCommit` (`cmd/camp/commit.go`), after the staging block (`:137-154`) and before the commit (`:188`), add a guard that runs only when operating at the campaign root without `--include-refs`:

```go
// After staging and before committing, check for pre-staged gitlinks.
// A pre-staged projects/* gitlink at the campaign root means either:
//   (a) a previous camp commit/refs-sync was interrupted, or
//   (b) a concurrent fest/camp process staged a submodule pointer.
// Either way, committing it under this message tag is wrong.
if !target.IsSubmodule && !commitIncludeRefs {
    stagedRefs, err := listStagedProjectRefs(ctx, target.Path)
    if err != nil {
        return camperrors.Wrap(err, "check pre-staged refs")
    }
    if len(stagedRefs) > 0 {
        return camperrors.NewValidation("pre_staged_refs",
            fmt.Sprintf(
                "staged submodule ref(s) found that were not staged by this command: %s\n"+
                    "These are not committed by 'camp commit' without --include-refs.\n"+
                    "Options:\n"+
                    "  camp refs-sync          -- commit only the submodule pointers\n"+
                    "  camp commit --include-refs -m \"...\"  -- include them in this commit\n"+
                    "  git reset HEAD %s       -- unstage them to continue",
                strings.Join(stagedRefs, ", "),
                strings.Join(stagedRefs, " "),
            ), nil)
    }
}
```

`listStagedProjectRefs` is a new unexported helper in `cmd/camp/commit.go`:

```go
// listStagedProjectRefs returns staged paths under projects/ in the given repo.
// Used to detect pre-existing gitlink stages before a campaign-root commit.
func listStagedProjectRefs(ctx context.Context, repoPath string) ([]string, error) {
    cmd := exec.CommandContext(ctx, "git", "-C", repoPath,
        "diff", "--cached", "--name-only", "-z", "--", "projects")
    out, err := cmd.Output()
    if err != nil {
        return nil, camperrors.NewGit("diff --cached projects", "", "", "", err)
    }
    return parseNULList(out), nil
}

func parseNULList(out []byte) []string {
    raw := strings.TrimRight(string(out), "\x00")
    if raw == "" {
        return nil
    }
    return strings.Split(raw, "\x00")
}
```

Place the import for `os/exec` if not already present; it is already in the file via `camperrors`.

**Placement within runCommit:** insert the guard AFTER the `cmdutil.ShowStagedSummary` call (`:157`) and BEFORE the `hasChanges` check (`:160`). This ensures the guard only fires when there is staged content and only when the user did not explicitly opt in.

**Why not scope the commit instead of refusing:** scoping the final commit to "all paths `StageAllExcluding` just staged" would require either (a) returning a path list from `StageAllExcluding` (API change with wider blast radius) or (b) running `git diff --cached --name-only` after staging to enumerate what was staged (a second git call, and the window between that call and the commit is where concurrent staging could add paths). The refusal approach is simpler, visible, and explicit. The user can use `--include-refs` to opt in or `git reset HEAD projects/X` to unstage. Document the tradeoff in the code comment.

### Step 3: Refs-only-drift friendly hint

When `StageAllExcluding` stages nothing (because all changes are under `projects/`) and the final `executor.Commit` fails with "no changes added to commit", the current flow:

```
executor.Commit -> executeCommit -> ClassifyGitError -> GitErrorUnknown -> camperrors.NewGit(...)
```

After step 1's fix, this becomes:

```
executor.Commit -> executeCommit -> ClassifyGitError -> GitErrorNoChanges -> ErrNoChanges
```

Which hits the existing `errors.Is(err, git.ErrNoChanges)` check at `:195` and prints "Nothing to commit". This is close but the user deserves a hint about the refs situation. Add a hint specifically when the campaign root has refs-only drift:

```go
// In runCommit, at the ErrNoChanges handling block:
if errors.Is(err, git.ErrNoChanges) {
    if !target.IsSubmodule && !commitIncludeRefs {
        // Check if refs-only drift explains the empty commit.
        driftPaths, _ := listStagedProjectRefs(ctx, target.Path)
        if len(driftPaths) > 0 {
            fmt.Println(ui.Warning("Nothing to commit (submodule ref changes are excluded by default)"))
            fmt.Println(ui.Dim("  Use 'camp refs-sync' to commit only the submodule pointers"))
            fmt.Println(ui.Dim("  Use 'camp commit --include-refs -m \"...\"' to include them"))
            return nil
        }
    }
    fmt.Println(ui.Success("Nothing to commit"))
    return nil
}
```

This mirrors the existing hint in `stage.go` that the review references.

### Tests

Add to `cmd/camp/commit_test.go` (host `t.TempDir()`, existing convention):

**Test 1: pre-staged gitlink is refused without --include-refs**

```
setup:
  campaign root with a submodule at projects/foo
  manually stage the gitlink: git -C campRoot add projects/foo

call runCommit (via the cobra command or directly) without --include-refs
assert: error message contains "staged submodule ref(s)" and "projects/foo"
assert: git diff --cached at campRoot still shows projects/foo staged (not committed, not unstaged)
```

**Test 2: refs-only drift produces the friendly hint, not a raw error**

```
setup:
  campaign root; StageAllExcluding stages nothing because only projects/ has changes
  (simulate: stage nothing, then call executor.Commit)

mock or call ClassifyGitError directly with stderr "no changes added to commit":
assert: ClassifyGitError returns GitErrorNoChanges

integration-level:
  campaign root with one submodule ahead; no other content changes
  run camp commit -m "test"
  assert: exit 0 (not an error)
  assert: output contains "refs-sync" hint or "submodule ref changes are excluded"
```

## Done When

- [ ] All requirements met
- [ ] `ClassifyGitError` matches both "nothing to commit" and "no changes added to commit" for `GitErrorNoChanges`
- [ ] `runCommit` refuses pre-staged `projects/*` gitlinks at the campaign root without `--include-refs`, naming the paths and offering recovery options
- [ ] Refs-only-drift case prints the friendly hint pointing to `camp refs-sync`
- [ ] Test 1 (pre-staged gitlink refused) passes
- [ ] Test 2 (refs-only drift hint) passes, at least at the `ClassifyGitError` unit level
- [ ] Both-profile gate passes (`just build`, `BUILD_TAGS=dev just build`, `just test unit`, `go test -tags dev ./...`, `just lint`, `BUILD_TAGS=dev just lint`)

## Out of Scope

- Scoping the final commit to exactly-staged paths (the refusal approach is chosen; document this tradeoff in the code comment at the guard site).
- Changes to `StageAllExcluding` behavior or return type (would have wide blast radius; sequence 11 is the right place for API unification).
- `--amend` behavior: `runCommit --amend` skips the staging block; the pre-staged gitlink guard is inside the staging-conditional block and should also apply to the amend path if it uses the same final commit. Verify and add a guard for the amend path if needed.