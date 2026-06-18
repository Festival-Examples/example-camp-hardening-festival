---
fest_type: task
fest_id: 02_promote_ingest_boundary.md
fest_name: promote_ingest_boundary
fest_parent: 05_fest_boundary
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.675946-06:00
fest_updated: 2026-06-14T21:52:04.670096-06:00
fest_tracking: true
---


# Task: Promote Ingest Boundary

## Objective

Stop `copyIntentToIngest` from MkdirAll-ing fest-internal festival ingest directories, surface fest CLI stderr on promote failure, and create a fest workitem capturing the missing `fest ingest <file>` capability (WF-7 from findings-subsystems.md).

## Requirements

- [x] `copyIntentToIngest` in `internal/intent/promote/promote.go:344-359` does NOT call `os.MkdirAll` when `001_INGEST/input_specs` is absent; instead it skips the copy and writes a clear notice to stderr telling the user to ingest the file via fest.
- [x] `createFestival` in `internal/intent/promote/promote.go:293-312` surfaces `exec.ExitError.Stderr` in the returned `Result` when the fest CLI exits non-zero; callers that display the result to users show this stderr text.
- [x] During execution the executor creates a design workitem in the fest project using `camp workitem create` with `--type design`, describing the missing `fest ingest <file>` capability and referencing WF-7 and this festival (CH0001).
- [x] Unit test `TestCopyIntentToIngest_SkipsAbsentIngestDir`: calling `copyIntentToIngest` when the `001_INGEST/input_specs` directory does not exist does not create the directory, does not create any files under `festivals/`, and returns false (or the skip-notice variant of the result, depending on how the function is refactored).
- [x] Unit test `TestCreateFestival_SurfacesStderr`: when the fest CLI exits non-zero, the returned `Result` contains the stderr text from the subprocess.

## Implementation

### Background: the "consumer reaches behind the tool" anti-pattern

When `camp intent promote` creates a festival, it correctly shells out to `fest create festival --json` (line 307 in the worktree at `bc2ad1f`). That delegation is right. The problem is what happens next: `copyIntentToIngest` (line 344) constructs the path `festivals/<dest>/<festival>/001_INGEST/input_specs` and calls `os.MkdirAll` to create it if it is absent (lines 351-354), then copies the intent file into it.

This is the "consumer reaches behind the tool" pattern. The `001_INGEST/input_specs` directory is part of fest's internal festival layout. If fest changes its ingest layout (for example, renaming the directory or restructuring the phase), camp silently writes orphan files into whatever path its hardcoded string resolves to. The files may not be recognized by fest, and there is no way for camp to know they were not. The correct surface would be a `fest ingest <file>` command that camp can shell out to, letting fest decide where and how to receive the file.

Since that command does not exist yet (the workitem this task creates tracks it), the fix is to stop MkdirAll-ing: if the ingest directory already exists (because fest created the festival and the directory is part of fest's real layout), copy the file in; if it does not exist, skip and tell the user what to do. This is safer than creating orphan files: if fest already created the directory, the copy is almost certainly correct; if it did not, there is no safe destination.

The second problem (fest stderr discarded) compounds this: when `cmd.Output()` fails at line 309-312, the code returns `Result{FestivalName: festivalName}` with no indication of why fest exited non-zero. The user sees a generic partial-success message. Surfacing the stderr text gives them the actual cause (fest not initialized, wrong permissions, etc.).

### Step 1: Modify `copyIntentToIngest` to skip when the directory is absent

File: `internal/intent/promote/promote.go`

Verified function location at `bc2ad1f`: `copyIntentToIngest` is at lines 344-359. The `os.MkdirAll` block is at lines 351-354.

Replace the function body so that the MkdirAll branch becomes a skip-with-notice:

```go
func copyIntentToIngest(campaignRoot, dest, festivalDir string, i *intent.Intent) bool {
    if i.Path == "" {
        return false
    }

    ingestDir := filepath.Join(campaignRoot, "festivals", dest, festivalDir, "001_INGEST", "input_specs")

    if _, err := os.Stat(ingestDir); os.IsNotExist(err) {
        fmt.Fprintf(os.Stderr,
            "Notice: intent file not copied to festival ingest directory because %s does not exist.\n"+
                "To ingest the file manually, run: fest (ingest command not yet available; see workitem for tracking)\n"+
                "File to ingest: %s\n",
            ingestDir, i.Path)
        return false
    } else if err != nil {
        return false
    }

    destPath := filepath.Join(ingestDir, filepath.Base(i.Path))
    return copyFile(i.Path, destPath) == nil
}
```

The `fmt` import is already present in the file (line 15 in the worktree). The `os` import is at line 17. No new imports are needed.

The notice text tells the user exactly what to do once `fest ingest` exists. It avoids creating any files under `festivals/`. When the directory already exists (fest created it as part of the festival scaffold), the copy proceeds as before.

### Step 2: Surface fest CLI stderr on promote failure

File: `internal/intent/promote/promote.go`

Verified location at `bc2ad1f`: `createFestival` at lines 293-339. The `cmd.Output()` call is at line 309. The error branch at lines 310-312 returns `Result{FestivalName: festivalName}` with no error detail.

The `exec.ExitError` type carries a `Stderr []byte` field when the command was run with `cmd.Output()` (Go's `exec.Cmd.Output` captures stderr into `ExitError.Stderr` automatically). Surface it by adding an `ErrorDetail` field to `Result` or by reusing an existing field. Check the existing `Result` struct definition (lines 50+ in the file) to pick the right field.

```go
output, err := cmd.Output()
if err != nil {
    result := Result{FestivalName: festivalName}
    var exitErr *exec.ExitError
    if errors.As(err, &exitErr) && len(exitErr.Stderr) > 0 {
        result.FestCLIError = strings.TrimSpace(string(exitErr.Stderr))
    }
    return result
}
```

This requires adding a `FestCLIError string` field to the `Result` struct if it does not already exist. Read the `Result` struct at the top of `promote.go` (around line 50) before editing. If a suitable field already exists (e.g., an error detail string), use it. If not, add `FestCLIError string` to the struct.

Also add `"errors"` to the import block if it is not already present (it may already be there; check before adding). The `strings` package is already imported (line 21 in the worktree).

Then ensure callers that display the result to users check and print `FestCLIError`. The two callers are:
- `cmd/camp/intent/promote.go` (the CLI command): find where it prints the result to the user and add a check.
- `internal/intent/tui/explorer*.go` or equivalent TUI promote action: find where it renders the result and add the stderr text to the output.

To locate the callers: run `grep -rn "promote\.\|Result{" cmd/camp/intent/promote.go` in the worktree. Read each caller to understand how it uses `Result` and add a block like:

```go
if result.FestCLIError != "" {
    fmt.Fprintf(os.Stderr, "fest error: %s\n", result.FestCLIError)
}
```

### Step 3: Create the fest workitem during execution

This step happens at execution time, not in Go source code. The executing agent must run the following command from the campaign root (not from the worktree, since the workitem is about the fest project):

First, check the available flags:

```bash
camp workitem create --help
```

Then create the workitem. The command to run looks like this (adjust `--type` and `--title` based on what `--help` shows as valid values; `design` is the correct type for a capability gap that needs architectural design before implementation):

```bash
camp workitem create \
    --type design \
    --title "fest ingest command: accept a file into festival 001_INGEST/input_specs" \
    fest-ingest-capability
```

Inside the created workitem directory, add a `.workitem` note explaining:
- WF-7: camp's `copyIntentToIngest` hardcodes `festivals/<dest>/<festival>/001_INGEST/input_specs` and previously MkdirAll-ed it.
- The missing capability: `fest ingest <file> [--festival <name>]` that accepts a file and places it into the correct ingest location for the named or current festival.
- Reference: this is tracked by festival camp-hardening-CH0001, task 02_promote_ingest_boundary.
- Once this command exists, `camp intent promote` should shell out to it rather than hardcoding the path.

If `camp workitem create` is not available from your session context (the worktree does not have it in its own `just` recipes since it is the camp source tree, not a campaign), run the command from the campaign root directory using `cr just workitem create ...` or navigate to the campaign root and run it there. Do not modify the fest source repository directly.

### Step 4: Write unit tests

File: `internal/intent/promote/promote_test.go` (existing)

Use `t.TempDir()` for the campaign fixture. Follow the existing test structure.

```go
func TestCopyIntentToIngest_SkipsAbsentIngestDir(t *testing.T) {
    root := t.TempDir()
    // Create the festival directory up to the level fest would create,
    // but intentionally leave 001_INGEST/input_specs absent.
    festDir := filepath.Join(root, "festivals", "planning", "my-festival-abc123")
    if err := os.MkdirAll(festDir, 0755); err != nil {
        t.Fatal(err)
    }
    // Create a fake intent file to copy.
    intentPath := filepath.Join(root, "intent.md")
    if err := os.WriteFile(intentPath, []byte("# Test"), 0644); err != nil {
        t.Fatal(err)
    }
    i := &intent.Intent{Path: intentPath}

    result := copyIntentToIngest(root, "planning", "my-festival-abc123", i)

    if result {
        t.Error("expected copyIntentToIngest to return false when ingest dir is absent")
    }
    // Confirm nothing was created under festivals/.
    entries, err := os.ReadDir(festDir)
    if err != nil {
        t.Fatal(err)
    }
    if len(entries) != 0 {
        t.Errorf("expected no files created under festival dir, got: %v", entries)
    }
    absent := filepath.Join(festDir, "001_INGEST")
    if _, err := os.Stat(absent); !os.IsNotExist(err) {
        t.Errorf("001_INGEST should not have been created, but stat gave: %v", err)
    }
}
```

For `TestCreateFestival_SurfacesStderr`, the challenge is that `createFestival` calls `fest.FindFestCLI()` to locate the fest binary. In a test environment the fest binary may not be installed. The cleanest approach without adding a test injection point is to test the stderr surfacing logic by verifying the `Result.FestCLIError` field is populated when `exec.ExitError.Stderr` is non-empty. If `fest.FindFestCLI()` returns an error in the test environment, the test will hit the `FestNotFound: true` branch before reaching the stderr logic. In that case, write the test for the stderr extraction logic separately, or add a thin helper function `extractFestStderr(err error) string` that the test can call directly:

```go
// extractFestStderr returns the stderr text from an exec.ExitError, or empty string.
func extractFestStderr(err error) string {
    var exitErr *exec.ExitError
    if errors.As(err, &exitErr) && len(exitErr.Stderr) > 0 {
        return strings.TrimSpace(string(exitErr.Stderr))
    }
    return ""
}
```

Then test `extractFestStderr` directly with a synthetic `exec.ExitError`:

```go
func TestCreateFestival_SurfacesStderr(t *testing.T) {
    // Use a command that exits non-zero with known stderr output.
    cmd := exec.Command("sh", "-c", "echo 'fest: festival already exists' >&2; exit 1")
    _, err := cmd.Output()
    if err == nil {
        t.Fatal("expected non-zero exit")
    }
    got := extractFestStderr(err)
    if got == "" {
        t.Error("expected stderr to be captured, got empty string")
    }
    if !strings.Contains(got, "fest: festival already exists") {
        t.Errorf("stderr text not found in result: %q", got)
    }
}
```

This test is host-safe, uses no filesystem mutation, and verifies the core extraction logic regardless of whether fest is installed. Add it to `promote_test.go` alongside the existing tests.

## Edge Cases

- `copyIntentToIngest` called when `i.Path` is empty: already handled by the existing early return at line 345 in the worktree. No change needed there.
- `ingestDir` exists but is a file (not a directory): the `os.Stat` check returns no error, and `copyFile` will fail when it tries to write into a path that is a file name, not a directory. This is the same behavior as before the fix and is acceptable.
- fest not installed: `createFestival` already returns `Result{FestNotFound: true}` when `fest.FindFestCLI()` fails (line 295-296 in the worktree). The FestNotFound path is taken before reaching the `cmd.Output()` call, so no stderr extraction is needed there.
- Symlinked campaign roots: `copyIntentToIngest` constructs the path using `filepath.Join(campaignRoot, "festivals", ...)` where `campaignRoot` is whatever was passed in. If the caller passes a symlinked path, the constructed path may not resolve to the real filesystem location. This is a pre-existing issue not introduced by this fix; the fix does not make it worse.

## Out of Scope

- Implementing `fest ingest <file>`: that is the fest project's responsibility, tracked by the workitem created in step 3.
- Modifying the fest source repository: per constraint C-5, fest-side changes are out of scope for this festival.
- Changing how `createFestival` constructs the festival path or shells to fest: the shell-out is correct. Only the post-creation ingest copy and the error surfacing are in scope.

## Done When

- [x] `copyIntentToIngest` at `internal/intent/promote/promote.go` does not call `os.MkdirAll` under any code path; the absent-directory branch prints a notice and returns false.
- [x] `createFestival` at `internal/intent/promote/promote.go` populates a `FestCLIError` (or equivalent) field in the returned `Result` when the fest CLI exits non-zero; callers that print results to users display this field.
- [x] `TestCopyIntentToIngest_SkipsAbsentIngestDir` passes: no files created under `festivals/`, function returns false.
- [x] `TestCreateFestival_SurfacesStderr` passes: stderr text from a non-zero subprocess is captured and returned.
- [x] A fest design workitem for `fest ingest <file>` exists in the campaign's workflow directory (created via `camp workitem create`).
- [x] Both-profile gate commands pass: `just build`, `BUILD_TAGS=dev just build`, `go test -count=1 -short ./internal/intent/promote/...` (stable and dev tags), `just lint`, `BUILD_TAGS=dev just lint`.

## Verification

- `go test -count=1 -short ./internal/intent/promote/...`
- `BUILD_TAGS=dev go test -count=1 -short ./internal/intent/promote/...`
- `go test -count=1 -short ./cmd/camp/intent ./internal/intent/tui/explorer`
- `just build stable`
- `just build dev`
- `just lint`
- `BUILD_TAGS=dev just lint`
- `git diff --check`

Project commit: `91a02a1` (`fix: stop writing fest ingest internals`).

Campaign workitem: `WI-64af81` at `workflow/design/fest-ingest-capability`.