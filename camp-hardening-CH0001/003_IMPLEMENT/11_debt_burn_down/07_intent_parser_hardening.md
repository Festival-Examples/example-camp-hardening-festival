---
fest_type: task
fest_id: 07_intent_parser_hardening.md
fest_name: intent_parser_hardening
fest_parent: 11_debt_burn_down
fest_order: 7
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.788548-06:00
fest_updated: 2026-06-15T09:53:30.618215-06:00
fest_tracking: true
---


# Task: Intent Parser Hardening

## Objective

Fix all four medium-severity intent parser and write-path bugs: line-anchored frontmatter delimiters, unknown-key round-trip, O_EXCL on guarded writes, id-verified removal, and gather archive failure surfacing including the N-26 orphan cleanup.

## Requirements

- [ ] Frontmatter delimiter matching is line-anchored (not substring-based); `---` inside a YAML value does not cause silent data corruption
- [ ] Unknown frontmatter keys survive `Move`/`Rename`/`Edit` via `yaml.Node` round-trip OR the supported-key contract is documented explicitly (recommendation: `yaml.Node`)
- [ ] Every stat-then-write site uses `O_CREATE|O_EXCL` so concurrent creates fail on EEXIST rather than racing to the same path
- [ ] `removeAllCopies` verifies frontmatter `id` before deleting (no false-positive deletion of a different intent with the same filename)
- [ ] Gather: a failed metadata `Save` after `CreateDirect` cleans up the half-written intent (N-26 orphan fix)
- [ ] Gather: archive failures collected into `GatherResult.Errors` instead of silently continued
- [ ] Both-profile gate green

## Implementation

### Background

R33 and N-26 cover the intent parser hardening. The worktree is at `bc2ad1f`. The key files are `internal/intent/parse.go`, `internal/intent/service.go`, `internal/intent/helpers.go`, and `internal/intent/gather/gather.go`. Verify line numbers before editing.

### Step 1: Line-anchored frontmatter delimiter

Location: `internal/intent/parse.go:104`. Current:

```go
var delimiter = []byte("---")
parts := bytes.SplitN(content, delimiter, 3)
```

Problem: `---` anywhere in the content (e.g., inside a YAML block scalar, inside markdown body) splits mid-content. The correct fix is line-anchored matching: `---` on its own line.

Replace the delimiter and parser:

```go
var frontmatterDelimiter = []byte("\n---\n")
var frontmatterStart = []byte("---\n")

func parseIntentFile(content []byte) (*parsedIntent, string, error) {
    // Content must start with "---\n"
    if !bytes.HasPrefix(content, frontmatterStart) {
        return nil, "", ErrInvalidFrontmatter
    }
    // Find the closing "\n---\n" delimiter
    rest := content[len(frontmatterStart):]
    idx := bytes.Index(rest, frontmatterDelimiter)
    if idx < 0 {
        return nil, "", ErrInvalidFrontmatter
    }
    frontmatter := rest[:idx]
    body := string(rest[idx+len(frontmatterDelimiter):])
    // ... unmarshal frontmatter YAML as before
```

The closing delimiter is `\n---\n` (newline-anchored on both sides). This means a `---` inside a YAML block scalar (which is indented and not on its own line) is never matched.

Test:
```go
func TestParseIntentDashInsideValue(t *testing.T) {
    content := `---
id: test-001
title: "My --- value"
---
Body text.
`
    intent, _, err := ParseIntentFile([]byte(content))
    if err != nil {
        t.Fatalf("parse failed: %v", err)
    }
    if intent.Title != `My --- value` {
        t.Errorf("title mangled: %q", intent.Title)
    }
}
```

Verify `SerializeIntent` produces `---\n` as the separator (it already does per `:159`; confirm the output still matches the new parser's expectations).

### Step 2: Unknown key round-trip via yaml.Node

Location: `internal/intent/parse.go:18-42` (`parsedIntent` struct with fixed schema) and `SerializeIntent` at `:150-174`.

Current: unknown frontmatter keys are dropped on `yaml.Unmarshal` into `parsedIntent` (fixed struct). Any user-added key is silently stripped on the next `Move`/`Rename`/`Edit`.

Fix via `yaml.Node`:

```go
type parsedIntent struct {
    // existing fields...
    extra yaml.Node // captures unknown keys for round-trip
}
```

The approach:
1. Unmarshal into a `yaml.Node` first
2. Decode known fields via node traversal
3. When serializing, re-encode: write known fields first, then any unknown key/value pairs from the original node

Practical implementation:

```go
func parseIntentFile(content []byte) (*Intent, string, error) {
    // Parse frontmatter bytes as yaml.Node
    var doc yaml.Node
    if err := yaml.Unmarshal(frontmatter, &doc); err != nil {
        return nil, "", ErrInvalidFrontmatter
    }
    // Map node to parsedIntent (walk the mapping node manually or use Decode)
    var p parsedIntent
    if err := doc.Decode(&p); err != nil {
        return nil, "", camperrors.Wrap(err, "decode frontmatter")
    }
    // Store the raw node for unknown-key preservation
    p.rawNode = &doc
    // ...
}
```

For `SerializeIntent`, marshal the known fields AND any unmapped keys from the raw node:

```go
func SerializeIntent(intent *Intent) ([]byte, error) {
    // Build output using yaml.Node to preserve unknown keys
    // 1. Start from intent.rawNode (if present)
    // 2. Update known-field values in the node
    // 3. Marshal the node
```

If implementing full `yaml.Node` round-trip is disproportionate for the scope of this task, document the supported-key contract explicitly instead:

Create `internal/intent/FRONTMATTER_CONTRACT.md`:
```
# Intent Frontmatter Contract

The following keys are officially supported and round-trip through Move/Rename/Edit:
id, title, status, created_at, type, concept, project (legacy), author, priority,
horizon, tags, blocked_by, depends_on, promotion_criteria, promoted_to,
gathered_from, gathered_at, gathered_into, updated_at.

Any other key is dropped on the next write operation. This is a known limitation
tracked at [issue/task reference]. Do not add unsupported keys to intent files.
```

**Recommendation:** implement `yaml.Node` round-trip. The intent subsystem is festival-app-facing; silently dropping user keys is a user-visible regression. The implementation is medium complexity (~50 lines) but provides a correct contract.

Test either way:
```go
func TestUnknownKeyRoundTrip(t *testing.T) {
    content := "---\nid: abc\ntitle: Test\ncustom_key: my value\n---\nBody\n"
    // Parse, then serialize, then parse again
    // Verify custom_key is present in the round-tripped output
}
```

### Step 3: O_EXCL on guarded write sites

Locations verified in worktree:
- `internal/intent/service.go:85-90`: `CreateDirect` does `os.Stat` then `os.WriteFile`
- `internal/intent/service.go:173`: `moveFile(tmpPath, finalPath)` in `CreateWithEditor` with no existence check on `finalPath`
- `internal/intent/service.go:317`: `os.WriteFile(newPath, data, 0644)` in `UpdateDirect` after a path derivation

For each site, replace the stat-then-write pattern with `O_CREATE|O_EXCL`:

**`CreateDirect` (`:85-90`):**
```go
// Before:
if _, err := os.Stat(finalPath); err == nil {
    return nil, camperrors.Wrap(ErrFileExists, finalPath)
}
if err := os.WriteFile(finalPath, []byte(content), 0644); err != nil {
// After:
f, err := os.OpenFile(finalPath, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
if err != nil {
    if os.IsExist(err) {
        return nil, camperrors.Wrap(ErrFileExists, finalPath)
    }
    return nil, camperrors.Wrap(err, "writing intent file")
}
defer f.Close()
if _, err := f.WriteString(content); err != nil {
    _ = os.Remove(finalPath)
    return nil, camperrors.Wrap(err, "writing intent content")
}
```

**`moveFile` in `helpers.go`:**
The cross-device fallback uses `os.Create` (O_TRUNC) which overwrites. Sequence 03 task 06 fixed this to gate on `EXDEV` specifically. Verify the sequence-03 fix is in place; if so, the `os.Create` in the fallback path is only reached on EXDEV. If sequence 03 has not landed, apply:

```go
// Fall back to copy + delete only on EXDEV
var linkErr *os.LinkError
if errors.As(err, &linkErr) && linkErr.Err == syscall.EXDEV {
    dstFile, createErr := os.OpenFile(dst, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0644)
    // ...
}
```

**`UpdateDirect`'s `os.WriteFile(newPath, ...)` (`:317`):**
Add an O_EXCL guard if `newPath` differs from the original path (status change moves the file):
```go
if newPath != intent.Path {
    // Moving to new path: use O_EXCL to avoid clobbering
    if _, err := os.Stat(newPath); err == nil {
        return nil, camperrors.Wrap(ErrFileExists, newPath)
    }
}
```
Then route through `fsutil.WriteFileAtomically` for the actual write (sequence 06 task 01 did this; verify).

Test:
```go
func TestCollisionSuffixLoopUnderConcurrentCreates(t *testing.T) {
    // Launch N goroutines calling CreateDirect with the same path target
    // Verify all but one get ErrFileExists or a suffixed path
    // Verify no data corruption
}
```

### Step 4: `removeAllCopies` id verification

Location: `internal/intent/service.go:390` (`removeAllCopies`). Current: deletes by matching filename without verifying that the frontmatter `id` matches the target id.

Fix: before removing, parse the file and check the id:

```go
func (s *IntentService) removeAllCopies(id string, exceptPath string) {
    for _, status := range AllStatuses() {
        dir := filepath.Join(s.intentsDir, string(status))
        entries, err := os.ReadDir(dir)
        if err != nil {
            continue
        }
        for _, entry := range entries {
            path := filepath.Join(dir, entry.Name())
            if path == exceptPath {
                continue
            }
            // Verify the frontmatter id before deleting
            candidate, err := ParseIntentFile(path)
            if err != nil || candidate.ID != id {
                continue // not the same intent
            }
            _ = os.Remove(path)
        }
    }
}
```

### Step 5: Gather archive failures and N-26 orphan cleanup

**N-26 orphan fix** (gather.go Save failure after CreateDirect):

Location: `internal/intent/gather/gather.go` around line 202 (`Save` call after creating the gathered intent). The current code:

```go
if err := s.intentSvc.Save(ctx, gathered); err != nil {
    return nil, camperrors.Wrap(err, "saving gathered intent")
}
```

The issue: `Save` is called after `CreateDirect` has already written the file. If `Save` fails (metadata update), the half-written intent remains on disk while the command errors. Fix with a cleanup:

```go
if err := s.intentSvc.Save(ctx, gathered); err != nil {
    // Clean up the half-written gathered intent to avoid orphans
    _ = os.Remove(gathered.Path)
    return nil, camperrors.Wrap(err, "saving gathered intent metadata (cleaned up partial file)")
}
```

**Archive failure collection** (gather.go:217-228):

Current: `continue` on both `Save` and `Move` errors with only a comment. The `GatherResult` struct at `:121-126` has `ArchivedPaths` and `ArchivedSources` but no error collection.

Add an `Errors` field:

```go
type GatherResult struct {
    Gathered        *intent.Intent
    SourceCount     int
    ArchivedPaths   []string
    ArchivedSources []*intent.Intent
    Errors          []error // archive errors (non-fatal; gathered intent was created)
}
```

Collect instead of silently continuing:

```go
if err := s.intentSvc.Save(ctx, src); err != nil {
    result.Errors = append(result.Errors, camperrors.Wrapf(err, "save source %s metadata", src.ID))
    continue
}
archived, err := s.intentSvc.Move(ctx, src.ID, intent.StatusArchived)
if err != nil {
    result.Errors = append(result.Errors, camperrors.Wrapf(err, "archive source %s", src.ID))
    continue
}
```

Callers of `Gather` should check `result.Errors` and warn if non-empty (they are non-fatal since the gathered intent was successfully created).

Test:
```go
func TestGatherFailureLeavesNoOrphan(t *testing.T) {
    // Set up a gather where Save fails after CreateDirect
    // Verify the gathered intent file is cleaned up
    // Verify no orphan remains on disk
}

func TestGatherArchiveFailuresCollected(t *testing.T) {
    // Set up sources where Move fails for one intent
    // Verify Gather returns result.Errors non-empty
    // Verify the gathered intent itself was created successfully
}
```

## Done When

- [ ] `TestParseIntentDashInsideValue` passes (`---` inside a YAML value does not corrupt the parse)
- [ ] `TestUnknownKeyRoundTrip` passes (unknown keys survive parse-serialize-parse cycle)
- [ ] `TestCollisionSuffixLoopUnderConcurrentCreates` passes
- [ ] `removeAllCopies` verifies frontmatter id before deleting (grep `removeAllCopies` and confirm the id check is present)
- [ ] `TestGatherFailureLeavesNoOrphan` passes
- [ ] `TestGatherArchiveFailuresCollected` passes
- [ ] Both-profile gate green