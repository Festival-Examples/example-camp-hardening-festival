---
fest_type: task
fest_id: 05_stage_vocabulary.md
fest_name: stage_vocabulary
fest_parent: 09_app_contracts
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.751348-06:00
fest_updated: 2026-06-15T01:06:37.614957-06:00
fest_tracking: true
---


# Task: Fix stage vocabulary, no-stage representation, and bump workitems to v1alpha6 (R26 / JSON-3)

## Objective

Introduce typed `LifecycleStage` constants with per-type valid sets, add an explicit no-stage representation so `--stage` filtering stops hiding design/explore items, publish per-type stage vocabulary in the JSON payload, include `ritual/` and `chains/` festivals in discovery, drop `omitempty` on the nested `WorkItemWorkflow` integer and boolean fields, bump the schema to `workitems/v1alpha6`, and create the festival-app coordination workitem per D004.

## Requirements

- [x] `LifecycleStage` is a typed string constant (or type with constants) in `internal/workitem/` with at minimum the stages surfaced today (`inbox`, `active`, `ready`, `planning`) plus a `LifecycleStageNone` sentinel for items that have no stage
- [x] Per-type valid stage sets are defined and consulted by `validateFlags` in `internal/commands/workitem/workitem.go`
- [x] `discover_workflow_docs.go` sets `LifecycleStage: LifecycleStageNone` (not `""`) for workflow-doc items; `filter.go` explicitly matches `LifecycleStageNone` when `--stage none` is requested
- [x] The JSON payload includes a `stage_vocabulary` map keyed by workflow type listing valid stages for that type
- [x] `ritual/` and `chains/` festival directories are included in `discoverFestivals` or explicitly documented as out-of-scope with a TODO comment in the file
- [x] `WorkItemWorkflow` fields `current_step`, `total_steps`, `completed_steps`, `blocked`, `doc_hash_changed` drop `omitempty` (JSON-3)
- [x] `internal/workitem/json.go` schema version constant is bumped from `workitems/v1alpha5` to `workitems/v1alpha6` with a changelog entry
- [x] A festival-app coordination workitem is created for the festival-app project listing every schema bump from D004 (see step 6 below)

## Implementation

### Background

`internal/workitem/discover_workflow_docs.go` (verified at worktree bc2ad1f) line 77: `LifecycleStage: ""` for workflow-doc items.
`internal/workitem/filter.go` line 21: `stageSet[item.LifecycleStage]` matches the empty string only when someone passes `--stage ""`, which is not a valid flag value. Result: design/explore items with no stage are invisible to `--stage` filters.
`internal/commands/workitem/workitem.go` line 230: `validStages := map[string]bool{"inbox": true, "active": true, "ready": true, "planning": true}`.
`internal/workitem/model.go` lines 57-64: `omitempty` on all `WorkItemWorkflow` int/bool fields.
`internal/workitem/json.go` line 16: `const SchemaVersion = "workitems/v1alpha5"`.
`internal/workitem/discover_festivals.go` line 34: discovery iterates only `planning`, `ready`, `active` stages; `ritual/` and `chains/` are not included.

Note: the `filter.go` type changed from `string` to `LifecycleStage` affects the `toSet` call signature. The `WorkItem.LifecycleStage` field type change propagates through all discovery callers; verify each site with a `grep -r "LifecycleStage" ./internal` before building.

### Step 1: define typed constants

In `internal/workitem/model.go` or a new `internal/workitem/stage.go`:

```go
// LifecycleStage identifies the workflow stage of a work item.
type LifecycleStage string

const (
    LifecycleStageNone     LifecycleStage = "none"
    LifecycleStageInbox    LifecycleStage = "inbox"
    LifecycleStageActive   LifecycleStage = "active"
    LifecycleStageReady    LifecycleStage = "ready"
    LifecycleStagePlanning LifecycleStage = "planning"
)

// StagesByType lists valid stages per workflow type.
var StagesByType = map[WorkflowType][]LifecycleStage{
    WorkflowTypeIntent:   {LifecycleStageInbox, LifecycleStageActive, LifecycleStageReady},
    WorkflowTypeDesign:   {LifecycleStageNone, LifecycleStageActive, LifecycleStageReady},
    WorkflowTypeExplore:  {LifecycleStageNone, LifecycleStageActive},
    WorkflowTypeFestival: {LifecycleStageNone, LifecycleStagePlanning, LifecycleStageReady, LifecycleStageActive},
}
```

Update `WorkItem.LifecycleStage` field type from `string` to `LifecycleStage`.

### Step 2: set `LifecycleStageNone` in discovery

In `internal/workitem/discover_workflow_docs.go` line 77, change:

```go
LifecycleStage: "",
```

to:

```go
LifecycleStage: LifecycleStageNone,
```

Do the same wherever festival items or other items are constructed without a known stage.

### Step 3: fix filter to match `LifecycleStageNone`

Update `filter.go` to handle the typed field:

```go
if len(stageSet) > 0 && !stageSet[string(item.LifecycleStage)] {
    continue
}
```

The `toSet` helper uses string keys; `stageSet["none"]` matches `LifecycleStageNone` items as long as the flag value `none` is accepted. Update `toSet` if needed to accept the string representation.

### Step 4: update `validateFlags`

In `internal/commands/workitem/workitem.go` line 230, add `"none"` to the valid set:

```go
validStages := map[string]bool{
    "none": true, "inbox": true, "active": true, "ready": true, "planning": true,
}
```

Update the error message string to include `none`.

### Step 5: add stage vocabulary to the JSON payload

In `internal/workitem/json.go`, add a `StageVocabulary` field to `Payload`:

```go
type Payload struct {
    SchemaVersion   string                `json:"schema_version"`
    GeneratedAt     time.Time             `json:"generated_at"`
    CampaignRoot    string                `json:"campaign_root"`
    Sort            SortInfo              `json:"sort"`
    Items           []WorkItem            `json:"items"`
    Counts          Counts                `json:"counts"`
    StageVocabulary map[string][]string   `json:"stage_vocabulary"`
}
```

In `NewPayload`, populate from `StagesByType`:

```go
vocab := make(map[string][]string, len(StagesByType))
for wt, stages := range StagesByType {
    strs := make([]string, len(stages))
    for i, s := range stages {
        strs[i] = string(s)
    }
    vocab[string(wt)] = strs
}
```

### Step 6: ritual and chains discovery

Open `internal/workitem/discover_festivals.go` and examine the directory layouts of `ritual/` and `chains/`. If they contain fest.yaml-based festival directories matching the same pattern as `planning`/`ready`/`active`, add them:

```go
for _, stage := range []string{"planning", "ready", "active", "ritual", "chains"} {
```

If their structure differs (e.g. chains entries are config files, not festival directories), add a comment:

```go
// TODO(R26): ritual/ and chains/ not included; their directory structure
// differs from planning/ready/active and requires separate discovery logic.
```

### Step 7: drop `omitempty` on `WorkItemWorkflow`

In `internal/workitem/model.go` lines 57-64, change the struct tags:

```go
CurrentStep    int    `json:"current_step"`
TotalSteps     int    `json:"total_steps"`
CompletedSteps int    `json:"completed_steps"`
RunStatus      string `json:"run_status,omitempty"`
Blocked        bool   `json:"blocked"`
DocHashChanged bool   `json:"doc_hash_changed"`
```

Keep `omitempty` on `RunStatus` (empty string is a valid "not running" distinction). The int and bool fields at 0/false are now distinguishable from absent.

### Step 8: bump schema version

In `internal/workitem/json.go`, update:

```go
// Changelog:
//   ...prior entries...
//   - v1alpha6: per-type stage vocabulary added (stage_vocabulary field);
//     LifecycleStageNone ("none") as explicit no-stage representation;
//     ritual/ and chains/ festivals included or documented;
//     omitempty dropped on WorkItemWorkflow int/bool fields (JSON-3).
const SchemaVersion = "workitems/v1alpha6"
```

### Step 9: create the festival-app coordination workitem

From the worktree root or the festival-app project directory:

```
camp workitem create --type design \
  --title "Update festival-app for camp contract changes (v1alpha6, intents/v1alpha1, profile field)" \
  --dir workflow/design
```

The workitem body (added to the created directory's primary doc) should list:
- `workitems/v1alpha5` -> `v1alpha6`: handle `stage_vocabulary`; update stage filter UI for `none`; handle non-omitempty ints/bools in `WorkItemWorkflow`
- `intents/v1alpha1`: new contract; app calls `--format json` (deprecated alias still emits new payload)
- `workflow-items/v1alpha1`: new contract; update app invocation
- `concepts/v1alpha1`: new contract (task 09.09)
- `version` adds `schema_version` and `profile` field; use `profile` to preflight dev-only commands
- `__manifest` version bumped; new annotated commands present

Record the workitem ID in this task file once created.

Created workitem ID: `design-festival-app-camp-contract-changes-2026-06-15`

### Tests

Update `internal/workitem/filter_test.go`:
- `--stage none` returns design/explore items with `LifecycleStageNone`, excludes inbox/active/ready items.
- `--stage active` excludes items with `LifecycleStageNone`.

Update `internal/workitem/json_test.go`:
- Assert `schema_version == "workitems/v1alpha6"`.
- Assert `stage_vocabulary` key is present and the `intent` entry includes `inbox`.

## Done When

- [x] All requirements met
- [x] `LifecycleStage` typed with `LifecycleStageNone` constant; `WorkItem.LifecycleStage` is typed
- [x] `--stage none` returns design/explore items; `--stage inbox` excludes them
- [x] JSON payload contains `stage_vocabulary` map
- [x] Schema version is `workitems/v1alpha6`
- [x] `WorkItemWorkflow` int/bool fields have no `omitempty`
- [x] ritual/chains included or documented out with TODO
- [x] Festival-app coordination workitem created and ID noted
- [x] Both-profile gate passes
- [x] Filter and JSON payload tests updated
