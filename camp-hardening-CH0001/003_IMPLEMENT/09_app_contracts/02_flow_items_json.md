---
fest_type: task
fest_id: 02_flow_items_json.md
fest_name: flow_items_json
fest_parent: 09_app_contracts
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.750654-06:00
fest_updated: 2026-06-15T00:38:59.472937-06:00
fest_tracking: true
---


# Task: Implement `camp flow items --json` versioned output (N-7)

## Objective

Replace the dead `--json` flag on `flow items` with a real versioned payload emitted through `jsoncontract`, move errors to stderr with non-zero exit, and use `cmd.Context()` throughout, so festival-app can parse flow item listings reliably.

## Requirements

- [x] `camp flow items --json` emits a versioned JSON payload to stdout with a `schema_version` field, a `generated_at` timestamp, and an `items` array; exit 0 on success
- [x] Errors during listing emit the standard jsoncontract error envelope to stderr and exit non-zero (not stdout, not exit 0)
- [x] The command uses `cmd.Context()` instead of `context.Background()`
- [x] `ListOptions.JSON` and `HistoryOptions.JSON` dead fields in `internal/workflow/service.go` are removed or implemented; if removing, confirm no other callers set them
- [x] The schema version string follows D004 (either a new `workflow-items/v1alpha1` or folded into the existing `workflow/v1` family if the shape is compatible; read both `internal/commands/workflow/` and the existing `workflow/v1` payload to decide); document the choice in a comment
- [x] `flow` is a dev-only command (`//go:build dev` gated via `internal/commands/release/register_dev.go`); the `items.go` file itself does not carry a build tag but compiles in both profiles; tests must not break the stable build gate

## Implementation

### Background

`internal/commands/flow/items.go` (verified at worktree bc2ad1f):

- Line 33: `ctx := context.Background()` instead of `cmd.Context()`
- Lines 60-76: the loop calls `svc.List(...)` with `ListOptions{JSON: flowItemsJSON}` but nothing in `workflow.Service.List` reads `ListOptions.JSON`; errors are `fmt.Printf("Error listing %s: ...", ...)` to STDOUT with `continue` and the function returns `nil` (exit 0)
- Line 84: `--json` flag bound to `flowItemsJSON`, which feeds `ListOptions.JSON` but is never read

`internal/workflow/service.go`:
- Line 177: `ListOptions.JSON bool` field declared but never read in `List()`
- Line 190: `HistoryOptions.JSON bool` field declared but never read

festival-app bundles the dev-profile binary and calls `flow items` as part of its workflow surface. A dead `--json` and stdout errors make this call unparseable 100% of the time.

### Step 1: decide the schema version string

Read `internal/commands/workflow/` (specifically the commands that use `workflow/v1`) to check whether `ListResult.Items` carries the same shape as the human table rows. If the shapes align, fold into `workflow/v1` by using the same schema version constant. If shapes differ (flow items has different fields), define `WorkflowItemsSchemaVersion = "workflow-items/v1alpha1"` in a new or existing `internal/commands/flow/json_contract.go`.

Add the constant with a changelog comment:

```go
// WorkflowItemsSchemaVersion is the JSON contract for camp flow items --json.
//
// Changelog:
//   - workflow-items/v1alpha1: initial versioned contract (N-7 fix)
const WorkflowItemsSchemaVersion = "workflow-items/v1alpha1"
```

### Step 2: define the payload struct

In the same file, define the output shape. At minimum:

```go
type FlowItemsPayload struct {
    SchemaVersion string           `json:"schema_version"`
    GeneratedAt   time.Time        `json:"generated_at"`
    Items         []FlowStatusItem `json:"items"`
}

type FlowStatusItem struct {
    Status  string          `json:"status"`
    Entries []FlowItemEntry `json:"entries"`
}

type FlowItemEntry struct {
    Name    string    `json:"name"`
    IsDir   bool      `json:"is_dir"`
    ModTime time.Time `json:"mod_time"`
}
```

Adjust field names to match whatever `workflow.ListResult.Items` already contains.

### Step 3: rewrite `newItemsCommand`

Replace the RunE in `internal/commands/flow/items.go`:

```go
RunE: jsoncontract.RunE(schemaVersion, func() bool { return flowItemsJSON }, func(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()

    cwd, err := getCwd()
    if err != nil {
        return err
    }

    svc := workflow.NewService(cwd)
    if err := svc.LoadSchema(ctx); err != nil {
        return err
    }

    statuses := resolveStatuses(svc, flowItemsAll, args)

    if !flowItemsJSON {
        return renderItemsTable(cmd, svc, ctx, statuses, flowItemsAll)
    }

    return renderItemsJSON(cmd, svc, ctx, statuses)
}),
```

`renderItemsJSON` collects all status results and emits the `FlowItemsPayload` to `cmd.OutOrStdout()`. Any per-status error returns a typed error (let `jsoncontract.RunE` handle envelope rendering).

### Step 4: remove dead fields

In `internal/workflow/service.go`, remove `JSON bool` from `ListOptions` (line 177) and `HistoryOptions` (line 190). Run `grep -r "ListOptions{" ./internal` and `grep -r "HistoryOptions{" ./internal` to confirm no callers set those fields. If any do, remove the field assignments there too.

### Step 5: add tests

The `flow` package compiles in both profiles. Tests for flow commands exist (or can be added) without a build tag because the package itself is not build-tagged. Create `internal/commands/flow/items_test.go`:

- A table-driven test that invokes `newItemsCommand()` against a campaign fixture with a loaded schema.
- Assert `--json` output is parseable with a `schema_version` field equal to `WorkflowItemsSchemaVersion`.
- Assert that a listing error (e.g. fixture with missing schema) produces a non-zero exit and the error envelope on stderr.

Check `internal/commands/flow/` for any existing test helpers to understand fixture setup.

## Done When

- [x] All requirements met
- [x] `camp flow items --json` emits parseable JSON with `schema_version` to stdout; exit 0
- [x] A listing error exits non-zero and emits the error envelope to stderr, nothing to stdout
- [x] `ListOptions.JSON` and `HistoryOptions.JSON` removed (or implemented) in `internal/workflow/service.go`
- [x] `cmd.Context()` used throughout; no `context.Background()` in the command
- [x] Both-profile gate passes
- [x] Tests cover JSON output shape and error envelope path
