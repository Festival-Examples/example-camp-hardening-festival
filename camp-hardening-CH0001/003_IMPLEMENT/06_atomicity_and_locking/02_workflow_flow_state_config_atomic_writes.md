---
fest_type: task
fest_id: 02_workflow_flow_state_config_atomic_writes.md
fest_name: workflow_flow_state_config_atomic_writes
fest_parent: 06_atomicity_and_locking
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.695479-06:00
fest_updated: 2026-06-14T22:23:59.231151-06:00
fest_tracking: true
---


# Task: Workflow, Flow, State, Config, and Scaffold Atomic Writes (WF-6 part 2, N-21, REG-3)

## Objective

Complete the atomic-write sweep by converting the remaining `os.WriteFile` and `os.Create`-then-write sites in workflow schema files, the flow registry, navigation state, config allowlist, and the `.gitignore` append helper; fix `LoadHistory` to skip corrupt lines rather than hard-failing; and delete the private `atomicWriteFile` clone in `create.go`.

## Requirements

- [x] **R11-WF6e**: `internal/workflow/service_init.go:58` (schema write) converted to `fsutil.WriteFileAtomically`.
- [x] **R11-WF6f**: `internal/workflow/service_sync_migrate.go:101` and `:221` (both schema writes) converted.
- [x] **R11-WF6g**: `internal/flow/registry.go:76` (flow registry save) converted.
- [x] **R11-WF6h**: `internal/state/location.go:105` (navigation state, `SaveEntry`): replace the `os.Create` + write-loop pattern with a buffer approach that writes atomically via `fsutil.WriteFileAtomically`.
- [x] **REG-3**: `internal/state/location.go` `LoadHistory` (currently `:61-63`): replace `return nil, fmt.Errorf(...)` on unmarshal failure with skip-and-warn; a single corrupt line must not prevent navigation state from loading.
- [x] **R11-WF6i**: `internal/scaffold/repair.go:469` (`appendGitignoreEntryIfMissing`): replace plain `os.WriteFile` with `fsutil.WriteFileAtomically` for the read-modify-write gitignore append.
- [x] **N-21**: `internal/config/allowlist.go:84` converted to `fsutil.WriteFileAtomically`.
- [x] **clone deletion**: Delete `atomicWriteFile` at `internal/commands/workitem/create.go:249-259`; switch its only direct caller (`:105`) to `fsutil.WriteFileAtomically`; confirm `doctor_ref_backfill.go:149` already uses `fsutil` for the same file type and cite it as the established pattern.
- [x] Existing test suites for all touched packages remain green in both profiles.
- [x] New tests: (a) a corrupt `state.jsonl` line no longer prevents `LoadHistory` from returning the valid entries; (b) each converted writer produces correct on-disk content after a round-trip read.
- [x] After this task, the grep checkpoint passes (see Done When).

## Implementation

### Background: why these sites matter

`fsutil.WriteFileAtomically` is already used for config, links, pins, and the nav cache (all verified clean in the second pass). The sites converted here are the remaining half of the split discipline. The workflow schema files (`.workflow.yaml`) are read by every `camp flow` subcommand and by festival-app; a torn write leaves every flow command in an error loop. The navigation state (`state.jsonl`) drives `cgo` toggle; a torn write plus the current hard-fail in `LoadHistory` bricks `cgo` until the file is manually cleared. The private `atomicWriteFile` clone in `create.go` is a weaker version (fixed `.tmp` suffix, no fsync) one import away from the real helper.

### Step 1: Verify line numbers against the worktree

The review baseline was `5c22d2b`/`11c270f`. The worktree is at `bc2ad1f`. Verify each site:

```bash
grep -n "os.WriteFile\|os.Create" \
  projects/worktrees/camp/camp-hardening/internal/workflow/service_init.go \
  projects/worktrees/camp/camp-hardening/internal/workflow/service_sync_migrate.go \
  projects/worktrees/camp/camp-hardening/internal/flow/registry.go \
  projects/worktrees/camp/camp-hardening/internal/state/location.go \
  projects/worktrees/camp/camp-hardening/internal/config/allowlist.go \
  projects/worktrees/camp/camp-hardening/internal/scaffold/repair.go \
  projects/worktrees/camp/camp-hardening/internal/commands/workitem/create.go
```

Expected results at `bc2ad1f` (may drift; relocate by function name if lines shifted):

| File | Line | Site |
|------|------|------|
| `workflow/service_init.go` | 58 | schema write in `InitWorkflow` |
| `workflow/service_sync_migrate.go` | 101 | v1 schema write in `SyncMigrate` |
| `workflow/service_sync_migrate.go` | 221 | v2 schema write in `MigrateV1ToV2` |
| `flow/registry.go` | 76 | registry save in `SaveRegistry` |
| `state/location.go` | 105 | `os.Create` in `SaveEntry` |
| `config/allowlist.go` | 84 | allowlist config write in `SaveAllowlistConfig` |
| `scaffold/repair.go` | 469 | gitignore append in `appendGitignoreEntryIfMissing` |
| `commands/workitem/create.go` | 249 | private `atomicWriteFile` function |

Note: `workflow/service_init.go` also has `os.WriteFile` at `:115` and `:181` for OBEY.md template writes. These write scaffold/template content, not schema state. Convert them as well for consistency: the cost is zero and any OBEY.md write mid-crash would leave a corrupt scaffold file.

Similarly, `workflow/service_sync_migrate.go:133` is an OBEY.md write inside the migrate loop. Convert it.

### Step 2: Add fsutil imports

Check for existing `fsutil` imports in each file:

```bash
grep -rn "fsutil" \
  projects/worktrees/camp/camp-hardening/internal/workflow/ \
  projects/worktrees/camp/camp-hardening/internal/flow/ \
  projects/worktrees/camp/camp-hardening/internal/state/ \
  projects/worktrees/camp/camp-hardening/internal/config/allowlist.go
```

Add `"github.com/Obedience-Corp/camp/internal/fsutil"` to each file that needs it.

### Step 3: Workflow schema writes

In `internal/workflow/service_init.go`, convert `:58`:

```go
// Before
if err := os.WriteFile(s.schemaPath, data, 0644); err != nil {
    return err
}

// After
if err := fsutil.WriteFileAtomically(s.schemaPath, data, 0644); err != nil {
    return err
}
```

Apply the same pattern to `:115` and `:181` if they are plain `os.WriteFile`.

In `internal/workflow/service_sync_migrate.go`, convert `:101`, `:133`, and `:221` using the same substitution.

### Step 4: Flow registry save

In `internal/flow/registry.go` at `:76`:

```go
// Before
if err := os.WriteFile(path, data, 0644); err != nil {
    return err
}

// After
if err := fsutil.WriteFileAtomically(path, data, 0644); err != nil {
    return err
}
```

### Step 5: Navigation state - fix LoadHistory and convert SaveEntry

**Fix `LoadHistory` to skip corrupt lines (REG-3):**

In `internal/state/location.go`, the current parse loop at approximately `:61-63` hard-fails:

```go
if err := json.Unmarshal([]byte(line), &entry); err != nil {
    return nil, fmt.Errorf("failed to parse line %d in state file: %w", lineNum, err)
}
```

Replace with skip-and-warn:

```go
if err := json.Unmarshal([]byte(line), &entry); err != nil {
    fmt.Fprintf(os.Stderr, "camp: state: skipping corrupt line %d in %s: %v\n", lineNum, stateFile, err)
    continue
}
```

This makes `cgo` toggle resilient to a torn write from a previous crash.

**Convert `SaveEntry` to use `fsutil.WriteFileAtomically` (REG-3, R11-WF6h):**

The current `SaveEntry` uses `os.Create(stateFilePath)` followed by a write loop. Replace with a buffer approach:

```go
// Replace the os.Create block starting at approximately :105
var buf bytes.Buffer
enc := json.NewEncoder(&buf)
for _, e := range entries {
    if err := enc.Encode(e); err != nil {
        return fmt.Errorf("failed to marshal entry: %w", err)
    }
}
if err := fsutil.WriteFileAtomically(stateFilePath, buf.Bytes(), 0600); err != nil {
    return fmt.Errorf("failed to write state file: %w", err)
}
```

Note: use `0600` to match security expectations for a per-machine cache file (or match the existing mode if it was `0644`; check the existing `os.Create` which creates with default 0666 umask-adjusted). The existing `os.Create` uses the process umask; use `0600` as the explicit default since this is a per-machine cache, not a shared config.

Add `"bytes"` to imports if not present; remove `"io"` or `"bufio"` imports that are no longer used.

### Step 6: Config allowlist write (N-21)

In `internal/config/allowlist.go` at `:84`:

```go
// Before
if err := os.WriteFile(configPath, data, 0644); err != nil {
    return camperrors.Wrap(err, "failed to write allowlist config")
}

// After
if err := fsutil.WriteFileAtomically(configPath, data, 0644); err != nil {
    return camperrors.Wrap(err, "failed to write allowlist config")
}
```

The `fsutil` import is already present in the `config` package (used by `registry.go`); confirm it is also in `allowlist.go`'s import block, or add it.

### Step 7: Gitignore append (R11-WF6i)

In `internal/scaffold/repair.go`, `appendGitignoreEntryIfMissing` at `:469`:

```go
// Before
return os.WriteFile(gitignorePath, append(raw, []byte(addition)...), 0o644)

// After
return fsutil.WriteFileAtomically(gitignorePath, append(raw, []byte(addition)...), 0o644)
```

This is a read-modify-write: `os.ReadFile` at `:457` then write at `:469`. The read and write are not locked, so a concurrent append can still race. The race window is narrow (only two `camp init --repair` processes both appending at the same time), and no lock is introduced here; the atomic write prevents partial content but not the race. Add a `TODO(seq06-lock)` comment if warranted, but this site is low-priority for locking.

### Step 8: Delete the private atomicWriteFile clone (R11-WF6j)

In `internal/commands/workitem/create.go`:

1. Remove the `atomicWriteFile` function at `:249-259`.
2. Switch the caller at `:105` to use `fsutil.WriteFileAtomically` directly:

```go
// Before
if err := atomicWriteFile(filepath.Join(target, ".workitem"), buf, 0o644); err != nil {

// After
if err := fsutil.WriteFileAtomically(filepath.Join(target, ".workitem"), buf, 0o644); err != nil {
```

3. Add the `fsutil` import to `create.go` if not already present.

The pattern to cite in the commit message or inline comment: `internal/commands/workitem/doctor_ref_backfill.go:149` already uses `fsutil.WriteFileAtomically` for the same `.workitem` file type. The private clone was one import away from the real thing, did not fsync, and used a fixed `.tmp` suffix (not random), making concurrent temp files collide.

### Step 9: Run the both-profile gate

```bash
just build
BUILD_TAGS=dev just build
go test -count=1 -short \
  ./internal/workflow/... \
  ./internal/flow/... \
  ./internal/state/... \
  ./internal/config/... \
  ./internal/scaffold/... \
  ./internal/commands/workitem/...
go test -tags dev -count=1 -short \
  ./internal/workflow/... \
  ./internal/flow/... \
  ./internal/state/... \
  ./internal/config/... \
  ./internal/scaffold/... \
  ./internal/commands/workitem/...
```

### Step 10: Add focused tests

**Test: corrupt state.jsonl line does not brick LoadHistory**

In `internal/state/` (create `location_test.go` if it does not exist):

```go
func TestLoadHistorySkipsCorruptLine(t *testing.T) {
    dir := t.TempDir()
    stateFile := filepath.Join(dir, ".campaign", "cache", "state.jsonl")
    if err := os.MkdirAll(filepath.Dir(stateFile), 0755); err != nil {
        t.Fatal(err)
    }

    valid := NavigationEntry{Location: "/good/path", Time: time.Now()}
    validJSON, _ := json.Marshal(valid)
    content := string(validJSON) + "\n" + "NOT_VALID_JSON\n"
    if err := os.WriteFile(stateFile, []byte(content), 0644); err != nil {
        t.Fatal(err)
    }

    entries, err := LoadHistory(context.Background(), dir)
    if err != nil {
        t.Fatalf("LoadHistory returned error on corrupt line: %v", err)
    }
    if len(entries) != 1 {
        t.Errorf("expected 1 valid entry, got %d", len(entries))
    }
    if entries[0].Location != "/good/path" {
        t.Errorf("unexpected entry: %v", entries[0])
    }
}
```

**Test: SaveEntry round-trips correctly after atomic rewrite**

```go
func TestSaveEntryRoundTrip(t *testing.T) {
    dir := t.TempDir()
    ctx := context.Background()

    entry := NavigationEntry{Location: "/test/path", Time: time.Now().Truncate(time.Second)}
    if err := SaveEntry(ctx, dir, entry); err != nil {
        t.Fatalf("SaveEntry: %v", err)
    }

    entries, err := LoadHistory(ctx, dir)
    if err != nil {
        t.Fatalf("LoadHistory: %v", err)
    }
    if len(entries) != 1 || entries[0].Location != entry.Location {
        t.Errorf("round-trip mismatch: got %v", entries)
    }
}
```

## Edge Cases

- **`bufio.Scanner` removal**: after converting `LoadHistory` from a scanner to a scanner-with-skip, ensure the `bufio.Scanner` is still used (just with `continue` instead of `return`). The scanner itself is fine; only the error handling changes.
- **OBEY.md writes in workflow**: `service_init.go:115,181` and `service_sync_migrate.go:133` write scaffold template content. These are create-once writes; converting them to atomic adds no functional change but ensures crash safety at no cost.
- **`bytes.Buffer` encoding**: `json.NewEncoder` appends a `\n` after each encoded value, which is what `json.Marshal` + manual `\n` did before. The resulting file format is identical.
- **`os` import in state/location.go**: after removing the `os.Create`/write-loop, verify that `os` is still imported for `os.MkdirAll`, `os.Open`, `os.IsNotExist`, and `os.Stderr`.

## Out of Scope

- `internal/workflow/service_init.go` OBEY.md writes beyond `:115` and `:181`: only the schema write at `:58` and the OBEY writes are in scope. Do not refactor the broader `InitWorkflow` function.
- Locking the gitignore append race: flagged with a `TODO(seq06-lock)` comment; not locked in this task.
- `internal/scaffold/init.go:326` (initial `.gitignore` creation, not append): this is a create-only write (no pre-existing file); it is lower risk than the append and is out of scope for this sequence. It is not part of the R11/WF-6 finding.
- `internal/commands/workitem/workitem.go:149` (path-output-to-file): writes a path string to a flag-specified file; not a state writer.
- Registry locking: `config.SaveRegistry` is already atomic; adding the lock is task 03, not this task.

## Done When

- [x] All requirements met.
- [x] `atomicWriteFile` function is gone from `internal/commands/workitem/create.go`.
- [x] Both-profile gate passes for all touched packages.
- [x] `LoadHistory` test with a corrupt line passes.
- [x] The grep checkpoint below shows only the expected remainder.

**Expected os.WriteFile remainder after this task** (for POLISH item C3 diff):

Run in the worktree root:
```bash
grep -rn "os.WriteFile" \
  internal/intent \
  internal/quest \
  internal/workflow \
  internal/flow \
  internal/state \
  internal/config \
  internal/scaffold \
  internal/commands/workitem
```

Expected hits (all legitimate non-state or migration writes):
- `internal/intent/migration_filesystem.go:91` - migration utility
- `internal/intent/migration.go:389` - migration utility
- `internal/intent/promote/promote.go:230` - promote README scaffold write
- `internal/intent/feedback/tracker.go:65` - feedback tracker
- `internal/intent/feedback/builder.go:44,56` - feedback builder (in-place rewrites of intent files; tracked under R33 sequence-11)
- `internal/scaffold/init.go:326` - initial `.gitignore` creation (create-only, not RMW)
- `internal/commands/workitem/workitem.go:149` - path output to flag file

Zero hits expected in: `internal/intent/service.go`, `internal/quest/quest.go`, `internal/workflow/service_init.go`, `internal/workflow/service_sync_migrate.go`, `internal/flow/registry.go`, `internal/config/allowlist.go`, `internal/scaffold/repair.go`, `internal/commands/workitem/create.go`.

## Completion Notes

- Project commits:
  - `b26a20c` (`fix: make workflow state writes atomic`)
  - `ffedd1b` (`test: cover gitignore append round trip`)
- Converted workflow schema and OBEY writes, flow registry save, navigation state save, allowlist save, scaffold `.gitignore` append, and workitem `.workitem` create/adopt writes to `fsutil.WriteFileAtomically`.
- Removed the private `atomicWriteFile` clone. The task text named `create.go` as the only caller, but `adopt.go` also called the clone in this worktree; it was converted to the shared helper too. `internal/commands/workitem/doctor_ref_backfill.go:149` was confirmed as the established shared-helper pattern for `.workitem` writes.
- Added focused tests:
  - `TestLoadHistorySkipsCorruptLine`
  - `TestSaveEntryRoundTrip`
  - `TestRunCreateWritesWorkitemMarker`
  - `TestAppendGitignoreEntryIfMissingRoundTrip`
- Verification:
  - `rg -n "os\\.WriteFile|os\\.Create|atomicWriteFile" internal/workflow/service_init.go internal/workflow/service_sync_migrate.go internal/flow/registry.go internal/state/location.go internal/config/allowlist.go internal/scaffold/repair.go internal/commands/workitem/create.go internal/commands/workitem/adopt.go || true`
  - `rg -n "atomicWriteFile" internal || true`
  - `go test -count=1 -short ./internal/workflow/... ./internal/flow/... ./internal/state/... ./internal/config/... ./internal/scaffold/... ./internal/commands/workitem/...`
  - `go test -tags dev -count=1 -short ./internal/workflow/... ./internal/flow/... ./internal/state/... ./internal/config/... ./internal/scaffold/... ./internal/commands/workitem/...`
  - `go test -v -count=1 -run 'TestLoadHistorySkipsCorruptLine|TestSaveEntryRoundTrip|TestRunCreateWritesWorkitemMarker|TestRunAdopt_WritesRef|TestSaveRegistry|TestSaveAllowlistConfig|TestService_Init' ./internal/state ./internal/commands/workitem ./internal/flow ./internal/config ./internal/workflow`
  - `go test -v -count=1 -run 'TestAppendGitignoreEntryIfMissingRoundTrip' ./internal/scaffold`
  - `go test -count=1 -short ./internal/scaffold/...`
  - `go test -tags dev -count=1 -short ./internal/scaffold/...`
  - `just build stable`
  - `just build dev`
  - `git diff --check`
  - `grep -rn --exclude='*_test.go' "os.WriteFile" internal/intent internal/quest internal/workflow internal/flow internal/state internal/config internal/scaffold internal/commands/workitem || true`

Note: the raw grep command in the task also reports numerous `*_test.go` fixture writes. The production-code checkpoint excluding tests matches the intended out-of-scope remainder, with zero hits in the converted target files.
