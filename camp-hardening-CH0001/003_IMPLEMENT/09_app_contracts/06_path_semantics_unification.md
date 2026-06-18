---
fest_type: task
fest_id: 06_path_semantics_unification.md
fest_name: path_semantics_unification
fest_parent: 09_app_contracts
fest_order: 6
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.751558-06:00
fest_updated: 2026-06-15T01:19:21.872331-06:00
fest_tracking: true
---


# Task: Unify path semantics across all JSON surfaces (N-16)

## Objective

Normalize every JSON surface that emits file paths to use campaign-relative paths plus a `campaign_root` field, with macOS `/private/var` symlink normalization applied once at the root boundary, so festival-app can join any path against `campaign_root` with a single rule.

## Requirements

- [ ] `camp intent list/find/count/show --json` items carry `campaign_root` (absolute, symlink-resolved) and `path` as campaign-relative (no leading `/`)
- [ ] `camp status all --json` carries a top-level `campaign_root` field; the campaign-root repo entry uses a campaign-relative path (`.` or empty string); submodule entries use campaign-relative paths as they already do
- [ ] Quest JSON output (dev-profile; no formal contract today) carries `campaign_root` and a campaign-relative `path`; note this in `docs/json-contracts.md` as best-effort
- [ ] `workitem` paths are already campaign-relative per `internal/workitem/model.go`; confirm the `campaign_root` field propagates correctly through the v1alpha6 payload built in task 09.05 (no change needed if already correct; document the verification)
- [ ] macOS symlink normalization: `filepath.EvalSymlinks` is called on `campaignRoot` exactly once in each command's setup path; the resolved root is used for all subsequent `filepath.Rel` calls so paths under `/var/...` and `/private/var/...` compare consistently
- [ ] `docs/json-contracts.md` documents the join rule: `abs_path = filepath.Join(campaign_root, relative_path)`

## Implementation

### Background

N-16 verified at worktree bc2ad1f:

- `internal/intent/parse.go` line 142: `intent.Path = path` where `path` is the absolute file path passed in. Intents emit this absolute path in `outputJSON`. On macOS, `os.TempDir()` returns `/tmp` which resolves to `/private/tmp`; a consumer comparing against the campaign root from `config.LoadCampaignConfigFromCwd` may get a different resolved form.
- `internal/quest/types.go` line 56: `Path string` carries the absolute path to the `quest.yaml` file.
- `cmd/camp/status_all.go` lines 113-114: the campaign-root `repoStatus` entry is constructed with `repoPath = campRoot` (absolute); line 169 overwrites `status.Path = p` where `p` is already relative for submodule entries, but the root entry keeps its absolute path in the JSON array.
- `internal/workitem/` is the model citizen: `internal/workitem/model.go` states "All path fields are campaign-relative" and `internal/workitem/json.go` emits `campaign_root` at the top level.

### Step 1: EvalSymlinks helper

Add a small helper in a suitable location (e.g. `internal/pathutil/` if that package exists, or `internal/config/`) so every command resolves the campaign root once:

```go
func ResolveRoot(root string) (string, error) {
    return filepath.EvalSymlinks(root)
}
```

Call this after loading `campaignRoot` from `config.LoadCampaignConfigFromCwd` in each affected command. If the function already exists elsewhere in the codebase, use it.

### Step 2: fix intent path output

In `cmd/camp/intent/json_contract.go` (created in task 09.04), the `IntentItem.Path` field should be populated as:

```go
resolvedRoot, err := filepath.EvalSymlinks(campaignRoot)
// ...
rel, err := filepath.Rel(resolvedRoot, intent.Path)
// use rel as the path; use resolvedRoot as campaign_root in the payload
```

Add `CampaignRoot string` to `IntentPayload` (already in task 09.04's design). Ensure it is populated with the symlink-resolved root.

The `IntentItem.Path` field in the existing inline `jsonIntent` struct (pre-task-09.04) carries the raw absolute path. After task 09.04 lands, this step just ensures the relative form is used in the new payload. If tasks are executed in order, verify the task 09.04 implementation used the relative form; if not, fix it here.

### Step 3: fix status all path output

In `cmd/camp/status_all.go`, the `statusAllJSON` output struct currently includes `repoStatus.Path` which for the campaign-root entry holds the absolute path. Fix:

1. Before emitting JSON, set the campaign-root entry's `Path` to `"."` (or empty string with documentation).
2. Add a `CampaignRoot string` field to the JSON output struct:

```go
type statusAllOutput struct {
    SchemaVersion string       `json:"schema_version"`
    Timestamp     time.Time    `json:"timestamp"`
    CampaignRoot  string       `json:"campaign_root"`
    Repos         []repoStatus `json:"repos"`
}
```

Use `filepath.EvalSymlinks(campRoot)` for `CampaignRoot`. Note: the full versioning of the status-all payload is task 10.01 scope; here only add `campaign_root` and fix the root-entry path.

### Step 4: fix quest path output

Quest is dev-only. In the quest command that emits JSON (find the relevant command in `cmd/camp/quest/`), resolve the path to campaign-relative using `filepath.Rel(resolvedRoot, quest.Path)`. Add `campaign_root` to the quest JSON output. No schema version bump needed (no contract exists today); note in `docs/json-contracts.md` as best-effort.

### Step 5: verify workitem paths

Run:

```bash
grep -n "campaign_root\|CampaignRoot\|RelativePath\|relative_path" \
  internal/workitem/json.go internal/workitem/model.go
```

Confirm that `campaign_root` in `Payload` (json.go line 23) is populated from the resolved root in `NewPayload`. If `NewPayload` receives `campaignRoot` as-is, the caller must EvalSymlinks before passing it. Check `internal/commands/workitem/workitem.go`'s call site and add `EvalSymlinks` there if missing.

### Step 6: document the join rule

In `docs/json-contracts.md`, add a section:

```markdown
## Path semantics

All path fields in JSON payloads are campaign-relative. To resolve to an absolute path:

    abs_path = filepath.Join(campaign_root, relative_path)

`campaign_root` is always symlink-resolved (`filepath.EvalSymlinks`). On macOS, `/tmp` -> `/private/tmp`; always use the `campaign_root` from the payload rather than resolving paths independently.

The `status all` root entry uses `.` as its relative path. Quest paths are best-effort (no formal contract).
```

### Tests

Add a test in each affected package that:

1. Creates a campaign fixture under a symlinked root (e.g. via `os.Symlink` on a temp dir).
2. Runs the JSON command.
3. Asserts that every `path` in the output can be `filepath.Join`'d against `campaign_root` to produce a valid absolute path that resolves to an existing file/directory.

The workitem family already has this guarantee documented; a regression test confirming it holds after the v1alpha6 bump is sufficient.

## Done When

- [ ] All requirements met
- [ ] Intent JSON uses campaign-relative paths with `campaign_root`
- [ ] Status all JSON has `campaign_root` field; root entry path is `"."`
- [ ] Quest JSON (dev profile) uses campaign-relative paths
- [ ] Workitem `campaign_root` is EvalSymlinks-resolved at the call site
- [ ] `docs/json-contracts.md` documents the join rule
- [ ] Tests verify path join on a symlinked-root fixture for at least one surface
- [ ] Both-profile gate passes