---
fest_type: task
fest_id: 04_intent_json_contract.md
fest_name: intent_json_contract
fest_parent: 09_app_contracts
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.751126-06:00
fest_updated: 2026-06-15T00:57:00.052336-06:00
fest_tracking: true
---


# Task: Version the intent JSON contract (R20 / JSON-1 / JSON-2 / N-15 / N-24)

## Objective

Replace all ad-hoc intent JSON emission with a versioned `intents/v1alpha1` envelope through `jsoncontract`, fix the multi-`--type` filter, resolve the `-f` flag collision on `intent add`, and add `docs/json-contracts.md` rows for intent and workitem priority surfaces.

## Requirements

- [x] `camp intent list --json`, `find --json`, `count --json`, and `show --json` each emit `{"schema_version":"intents/v1alpha1", "generated_at": ..., "campaign_root": ..., "items": [...]}` to stdout; the error envelope goes to stderr on failure; exit codes are correct
- [x] `--format json` (and `-f json`) remains accepted on all four commands as a deprecated alias that emits the SAME new payload (not the old bare array), so festival-app's current invocations keep working during transition
- [x] Multi-`--type` is honored: `camp intent list --type idea --type bug` returns items of both types (client-side filter, matching the multi-`--status` pattern already in the code)
- [x] `intent add` `-f` shorthand is moved off `--full` to eliminate the collision with `--format` on sibling commands; a new shorthand for `--full` must not conflict; leaving `--full` without a shorthand is acceptable
- [x] `camp intent add --json` emits a machine-readable success payload containing at minimum `{"schema_version": ..., "id": ..., "path": ...}` when the intent is created
- [x] `docs/json-contracts.md` gains rows for `camp intent list/find/count/show --json` (new `intents/v1alpha1`) and `camp workitem priority --json` (existing `workitem-priority/v1alpha1`, missing from the doc per JSON-1)
- [x] The scope statement separating contract surfaces from best-effort ones (JSON-4) is added to `docs/json-contracts.md`

## Implementation

### Background

`cmd/camp/intent/list.go` (verified at worktree bc2ad1f):
- Lines 136-140: `types[0]` only; multi-`--type` silently drops later values
- Lines 213-231: bare top-level array of inline `jsonIntent` structs, no `schema_version`, triggered via `--format json` not `--json`

`cmd/camp/intent/find.go` lines 166-173: same inline struct pattern, bare array.
`cmd/camp/intent/count.go` lines 102-107: inline structs, no envelope.
`cmd/camp/intent/add.go` line 81: `-f` bound to `--full` (TUI mode), clashing with `--format` shorthand on sibling commands.

The `internal/jsoncontract` package provides `RunE`, `Args`, `FlagErrorFunc`, and `Requested`. The workitem family uses this consistently. Mirror that pattern.

### Step 1: define the versioned contract

Create `cmd/camp/intent/json_contract.go`:

```go
package intent

import "time"

const IntentJSONVersion = "intents/v1alpha1"

// IntentPayload is the top-level JSON output for intent commands --json.
type IntentPayload struct {
    SchemaVersion string         `json:"schema_version"`
    GeneratedAt   time.Time      `json:"generated_at"`
    CampaignRoot  string         `json:"campaign_root"`
    Items         []IntentItem   `json:"items"`
}

// IntentItem is one intent in the JSON output.
type IntentItem struct {
    ID                string   `json:"id"`
    Title             string   `json:"title"`
    Type              string   `json:"type"`
    Status            string   `json:"status"`
    Concept           string   `json:"concept,omitempty"`
    Author            string   `json:"author,omitempty"`
    Priority          string   `json:"priority,omitempty"`
    Horizon           string   `json:"horizon,omitempty"`
    Tags              []string `json:"tags,omitempty"`
    BlockedBy         []string `json:"blocked_by,omitempty"`
    DependsOn         []string `json:"depends_on,omitempty"`
    PromotionCriteria string   `json:"promotion_criteria,omitempty"`
    PromotedTo        string   `json:"promoted_to,omitempty"`
    CreatedAt         string   `json:"created_at"`
    UpdatedAt         string   `json:"updated_at,omitempty"`
    Path              string   `json:"path"`
}
```

The `Path` field: this task fixes the path semantics as part of N-16 scope, but the shape change (campaign-relative + campaign_root) is task 09.06. For now, carry the absolute path as the existing code does; task 09.06 will migrate it. Do not mix scopes.

### Step 2: add `--json` flag and migrate each command

For each of `list.go`, `find.go`, `count.go`, `show.go`:

1. Add a `jsonOut bool` flag: `cmd.Flags().Bool("json", false, "emit a structured JSON result")`.
2. Keep `--format` flag; when `--format json` or `--json` is requested, both use the new envelope (check `jsonOut || format == "json"`).
3. Wrap RunE with `jsoncontract.RunE(IntentJSONVersion, func() bool { return jsonOut }, ...)`.
4. Replace the inline `outputJSON*` functions with a single helper that builds and marshals an `IntentPayload`.

Example pattern for `list.go`:

```go
var jsonOut bool
cmd.Flags().Bool("json", false, "emit a structured JSON result")

RunE: jsoncontract.RunE(IntentJSONVersion, func() bool { return jsonOut }, func(cmd *cobra.Command, args []string) error {
    // ... parse flags, load intents ...
    if jsonOut || format == "json" {
        return outputIntentPayload(cmd, campaignRoot, intents)
    }
    return outputTable(intents)
}),
```

`outputIntentPayload` marshals an `IntentPayload` and writes to `cmd.OutOrStdout()`.

### Step 3: fix multi-`--type` filter

In `buildListOptions` (`cmd/camp/intent/list.go` around line 136-140), apply all types client-side:

```go
// Before: uses only types[0]
// After:
typeSet := make(map[intent.Type]bool, len(types))
for _, t := range types {
    typeSet[intent.Type(t)] = true
}
// ... after fetching all intents, filter:
if len(typeSet) > 0 {
    filtered := intents[:0]
    for _, i := range intents {
        if typeSet[i.Type] {
            filtered = append(filtered, i)
        }
    }
    intents = filtered
}
```

Mirror the multi-`--status` pattern already implemented in the same file.

### Step 4: resolve `intent add` `-f` collision

In `cmd/camp/intent/add.go`, the flag at line 81 is:
```go
flags.BoolP("full", "f", false, "Full TUI mode with body textarea")
```

Remove the `-f` shorthand; leave the long form `--full` without a shorthand. No replacement shorthand is needed:

```go
flags.Bool("full", false, "Full TUI mode with body textarea")
```

### Step 5: add `--json` to `intent add`

After the intent is created, if `--json` was requested, emit:

```go
type intentAddResult struct {
    SchemaVersion string `json:"schema_version"`
    GeneratedAt   string `json:"generated_at"`
    ID            string `json:"id"`
    Path          string `json:"path"`
}
```

The `IntentJSONVersion` constant is reused. Write to `cmd.OutOrStdout()`. If the add triggers a TUI (full mode), skip JSON output and return an error if `--json` is set with `--full`.

### Step 6: update `docs/json-contracts.md`

Add rows:

| Surface | Schema version | Error envelope | Notes |
|---|---|---|---|
| `camp intent list/find/show --json` | `intents/v1alpha1` | yes | `--format json` is a deprecated alias |
| `camp intent count --json` | `intents/v1alpha1` | yes | counts inside `items[]` as status-count objects |
| `camp intent add --json` | `intents/v1alpha1` | yes | emits created id and path |
| `camp workitem priority --json` | `workitem-priority/v1alpha1` | yes | `cleared: true` when priority is cleared |

Add a scope statement section:

```markdown
## Scope: contract vs best-effort

Surfaces in this table are **versioned contracts**: schema_version will increment on breaking changes and festival-app keys on the version string.

Surfaces NOT in this table (e.g. `camp status all`, `camp version`, `camp __manifest`) are **best-effort**: they have JSON output but no formal version guarantee until explicitly promoted.
```

### Tests

Add `cmd/camp/intent/list_json_test.go` (and/or extend existing intent tests):

1. Golden payload shape: call `list --json` on a fixture campaign, unmarshal, assert `schema_version == "intents/v1alpha1"` and `items` is an array.
2. `--format json` emits the same payload as `--json` (alias equivalence).
3. Multi-type filter: fixture with intents of types `idea` and `bug`; `--type idea --type bug` returns both; `--type idea` returns only ideas.
4. `intent add --json` on a fixture: assert returned JSON has `id` and `path` fields.
5. Error path: list on a path that is not a campaign returns the error envelope to stderr, exit non-zero.

## Done When

- [x] All requirements met
- [x] `camp intent list/find/count/show --json` emit `intents/v1alpha1` envelope
- [x] `--format json` / `-f json` remain as deprecated aliases emitting the same payload
- [x] Multi-`--type` honored on `list`
- [x] `intent add` `-f` shorthand removed; `--json` emits created id/path
- [x] `docs/json-contracts.md` updated with intent rows, priority row, and scope statement
- [x] Both-profile gate passes
- [x] Tests cover version string, alias equivalence, multi-type filter, add output, error envelope
