---
fest_type: task
fest_id: 09_concepts_json.md
fest_name: concepts_json
fest_parent: 09_app_contracts
fest_order: 9
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.752502-06:00
fest_updated: 2026-06-15T01:48:47.305314-06:00
fest_tracking: true
---


# Task: Add versioned `--json` and fix not-in-campaign error for `camp concepts` (N-17)

## Objective

Give `camp concepts` a versioned `--json` output path with the standard error envelope, and fix the not-in-campaign case to exit non-zero with a typed error matching sibling commands.

## Requirements

- [ ] `camp concepts --json` emits a versioned JSON payload `{"schema_version":"concepts/v1alpha1", "generated_at": ..., "campaign_root": ..., "concepts": [...]}` when in a campaign
- [ ] `camp concepts` (or `camp concepts --json`) outside a campaign exits non-zero with a standard typed error, NOT `fmt.Println` + `return nil`
- [ ] The error envelope is emitted to stderr under `--json` when outside a campaign (standard `jsoncontract.RunE` behavior)
- [ ] The concept items in the JSON payload include at minimum `name`, `path`, `description`, `max_depth`, and the `has_items` flag as present in `concept.Concept`

## Implementation

### Background

`cmd/camp/concepts.go` (verified at worktree bc2ad1f) lines 28-34:

```go
cfg, campaignRoot, err := config.LoadCampaignConfigFromCwd(ctx)
if err != nil {
    fmt.Println(ui.Warning("Not in a campaign"))
    fmt.Println()
    fmt.Printf("Run %s to create a new campaign.\n", ui.Accent("camp init"))
    return nil
}
```

This prints to stdout and returns nil (exit 0). Every sibling command uses `return err` propagated through `camperrors` to exit non-zero. Festival-app calling `concepts` outside a campaign gets misleading exit 0 with human text on stdout.

There is no `--json` flag on `conceptsCmd` at all.

### Step 1: define the versioned contract

Add a constant in `cmd/camp/concepts.go` or alongside it:

```go
const ConceptsJSONVersion = "concepts/v1alpha1"
```

Define a payload struct:

```go
type conceptsPayload struct {
    SchemaVersion string          `json:"schema_version"`
    GeneratedAt   time.Time       `json:"generated_at"`
    CampaignRoot  string          `json:"campaign_root"`
    Concepts      []conceptItem   `json:"concepts"`
}

type conceptItem struct {
    Name        string   `json:"name"`
    Path        string   `json:"path"`
    Description string   `json:"description,omitempty"`
    MaxDepth    *int     `json:"max_depth,omitempty"`
    HasItems    bool     `json:"has_items"`
    Ignore      []string `json:"ignore,omitempty"`
}
```

### Step 2: add `--json` flag and wrap with `jsoncontract.RunE`

```go
var conceptsJSON bool

var conceptsCmd = &cobra.Command{
    Use:     "concepts",
    Short:   "List configured concepts",
    Aliases: []string{"concept"},
    RunE: jsoncontract.RunE(ConceptsJSONVersion, func() bool { return conceptsJSON }, runConcepts),
}

func init() {
    rootCmd.AddCommand(conceptsCmd)
    conceptsCmd.GroupID = "campaign"
    conceptsCmd.Flags().BoolVar(&conceptsJSON, "json", false, "emit a structured JSON result")
}
```

### Step 3: fix the not-in-campaign error

In `runConcepts`, replace the human-print-and-return-nil pattern with a proper error:

```go
cfg, campaignRoot, err := config.LoadCampaignConfigFromCwd(ctx)
if err != nil {
    return camperrors.Wrap(err, "not in a campaign directory")
}
```

When `--json` is set and this error occurs, `jsoncontract.RunE` will render the error envelope to stderr and return a non-zero `CommandError`. When `--json` is not set, cobra prints the error and exits non-zero as usual.

The human-friendly hint ("Run camp init to create a new campaign") can be attached via `jsoncontract.WithHint`:

```go
return jsoncontract.WithHint(
    camperrors.Wrap(err, "not in a campaign directory"),
    "run 'camp init' to create a new campaign",
)
```

### Step 4: emit JSON output

In `runConcepts`, after loading concepts:

```go
if conceptsJSON {
    resolvedRoot, _ := filepath.EvalSymlinks(campaignRoot)
    items := make([]conceptItem, len(concepts))
    for i, c := range concepts {
        items[i] = conceptItem{
            Name:        c.Name,
            Path:        c.Path,
            Description: c.Description,
            MaxDepth:    c.MaxDepth,
            HasItems:    c.HasItems,
            Ignore:      c.Ignore,
        }
    }
    payload := conceptsPayload{
        SchemaVersion: ConceptsJSONVersion,
        GeneratedAt:   time.Now().UTC(),
        CampaignRoot:  resolvedRoot,
        Concepts:      items,
    }
    enc := json.NewEncoder(cmd.OutOrStdout())
    enc.SetIndent("", "  ")
    return enc.Encode(payload)
}
printConcepts(cfg.Name, concepts)
return nil
```

Check the `concept.Concept` struct in `internal/concept/` to confirm the field names (`MaxDepth *int`, `HasItems bool`, etc.) match what `svc.List(ctx)` returns.

### Tests

Add `cmd/camp/concepts_test.go` or extend existing tests:

1. Outside-campaign: call `runConcepts` with no campaign fixture; assert non-zero return (error is non-nil).
2. Outside-campaign with `--json`: assert error envelope emitted to stderr and exit non-zero.
3. In-campaign with `--json`: assert stdout parses as JSON with `schema_version == "concepts/v1alpha1"` and `concepts` array present.

## Done When

- [ ] All requirements met
- [ ] `camp concepts --json` emits `concepts/v1alpha1` payload with concept list; exit 0
- [ ] `camp concepts` (with or without `--json`) outside a campaign exits non-zero
- [ ] Error envelope emitted to stderr under `--json` when outside a campaign
- [ ] Both-profile gate passes
- [ ] Tests cover outside-campaign error, JSON payload shape, and non-zero exit