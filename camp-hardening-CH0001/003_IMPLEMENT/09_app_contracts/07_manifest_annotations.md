---
fest_type: task
fest_id: 07_manifest_annotations.md
fest_name: manifest_annotations
fest_parent: 09_app_contracts
fest_order: 7
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.751785-06:00
fest_updated: 2026-06-15T01:33:28.944809-06:00
fest_tracking: true
---


# Task: Annotate every shipped command in the manifest (N-11)

## Objective

Add `agent_allowed` annotations to every shipped command that currently lacks one, split doctor's `--json` read surface from its `--fix` destructive surface in the annotation semantics, bump the manifest `version` field, and update `cmd/camp/manifest_test.go` counts coherently.

## Requirements

- [ ] Every command in the stable command tree carries an `agent_allowed` annotation; the manifest test's allowlist and count assertions pass in both profiles
- [ ] Every command in the dev command tree (dev profile build) also carries an annotation; the dev manifest test counts are consistent
- [ ] `doctor` annotation is updated: `agent_allowed: true` for the read path; the annotation or its reason documents that `--fix` is not allowed and an agent should never pass `--fix`; this is a documentation-only split (no runtime flag gating added here; actual runtime gating is sequence 10 scope)
- [ ] Commands currently missing annotations that are verified clean in `second-pass-fable.md`'s "hunted and found clean" list are annotated `agent_allowed: true` where applicable
- [ ] TUI-only commands (`intent explore`, `settings`, `intent crawl`, `dungeon crawl`) are annotated `agent_allowed: false` with reason "requires interactive terminal"
- [ ] The manifest `version` int field in `cmd/camp/manifest.go` is incremented from 1 to 2 (or to the current value plus 1 if already bumped by sequence 01)
- [ ] `cmd/camp/manifest_test.go` count assertions and the allowlist reflect the new annotations; check with sequence 01's changes already in place (sequence 01 added `workitem priority`)

## Implementation

### Background

`cmd/camp/manifest.go` (verified at worktree bc2ad1f):
- Line 12: `Version int` (not string)
- Lines 47-71: `walkCommands` only emits commands carrying an `"agent_allowed"` annotation key
- Result: only 44 commands appear in the dev manifest (43 in stable after sequence 01 restores the count)

`cmd/camp/doctor.go` lines 52-55:
```go
Annotations: map[string]string{
    "agent_allowed": "false",
    "agent_reason":  "Has --fix mode that is destructive",
},
```

The `--json` read path is safe for agents; only `--fix` is destructive. The annotation should reflect this so festival-app can call `camp doctor --json`.

N-11 lists the unannotated commands: `intent list/find/count/show/add`, `flow items`, `status all`, `workflow list/show`, `version`, `concepts`, and any others discovered by iterating the tree.

### Step 1: survey all unannotated commands

Run:

```bash
go run ./cmd/camp __manifest | jq '[.commands[].path] | length'
```

Then list every leaf command registered in `rootCmd` and compare against the manifest output to find gaps. Alternatively, grep for commands that have `RunE` or `Run` set but no `Annotations`:

```bash
grep -rn '"agent_allowed"' cmd/camp/ | sort
```

Produce a list of unannotated commands. For each one, decide:

- `agent_allowed: true`: reads only, no TUI, no destructive side effect
- `agent_allowed: false`: TUI required, or has destructive side effect that cannot be safely separated by flag

### Step 2: annotate missing commands

For each command, add:

```go
Annotations: map[string]string{
    "agent_allowed": "true",
    "agent_reason":  "Read-only listing; safe for agents",
},
```

or:

```go
Annotations: map[string]string{
    "agent_allowed": "false",
    "agent_reason":  "Requires interactive terminal",
    "interactive":   "true",
},
```

Key commands and their decisions:

| Command | Decision | Reason |
|---|---|---|
| `intent list/find/count/show` | `true` | read-only; has `--json` (task 09.04) |
| `intent add` | `false` (without `--json`); document that `--json` mode is safe | defaults to TUI without `--full`; see below |
| `flow items` | `true` | read listing; has `--json` (task 09.02) |
| `status all` | `true` | read-only status |
| `workflow list/show` | `true` | read-only |
| `version` | `true` | pure read |
| `concepts` | `true` | read-only |
| `intent explore` | `false` | TUI |
| `settings` | `false` | TUI |
| `intent crawl` | `false` | TUI |
| `dungeon crawl` | `false` | TUI |

For `doctor`, change the annotation:

```go
Annotations: map[string]string{
    "agent_allowed": "true",
    "agent_reason":  "Read path (--json) is safe; never pass --fix from an agent",
},
```

This acknowledges that the `--json` read surface is what festival-app calls. The comment in `agent_reason` documents the flag-level constraint.

For `intent add`: a simple annotation rule is to set `agent_allowed: true` with a reason noting that non-interactive use (body via `--body` or `--body-file`, not TUI) is safe; the `--full` flag is TUI-only and should not be passed by agents. The TTY guard added in sequence 10.07 will enforce non-TTY behavior at runtime.

### Step 3: bump manifest version

In `cmd/camp/manifest.go` line 12, the struct has `Version int`. After sequence 01 may have already changed the int; check the current value and increment by 1:

```go
manifest := Manifest{
    Version:  2,  // or 3 if sequence 01 already bumped it
    CLI:      "camp",
    Commands: []CommandEntry{},
}
```

Add a comment documenting the int-version convention vs. the string `schema_version` used elsewhere (JSON-4):

```go
// Version is an incrementing integer, not a semantic version string.
// This diverges from the schema_version convention used by other JSON surfaces
// (documented in docs/json-contracts.md JSON-4 scope statement).
Version int `json:"version"`
```

### Step 4: update manifest tests

`cmd/camp/manifest_test.go` has count and allowlist assertions. Sequence 01 already fixed the `workitem priority` gap. Now:

1. Run the test in both profiles to see the current counts.
2. Update the expected total-command count to match the new annotated count.
3. Update the `allowlist` map to include all newly annotated `agent_allowed: true` commands.
4. Verify that the `restricted` count (commands with `agent_allowed: false`) is also updated.

The test may have a structure like:
```go
const expectedStableAnnotated = 43 // update to new value
const expectedDevAnnotated = 44    // update to new value
```

Run `go test -run TestManifestCommand ./cmd/camp` in both profiles and iterate until green.

### Step 5: add a completeness test

Add a test that walks every registered command and asserts annotation presence:

```go
func TestManifestCoversAllCommands(t *testing.T) {
    root := newRootCmd() // or use rootCmd if accessible
    var unannotated []string
    walkForMissingAnnotations(root, "", &unannotated)
    if len(unannotated) > 0 {
        t.Errorf("commands missing agent_allowed annotation: %v", unannotated)
    }
}
```

This test prevents future regressions where a new command is added without an annotation.

## Done When

- [ ] All requirements met
- [ ] `camp __manifest` output in stable profile includes all stable commands
- [ ] `camp __manifest` output in dev profile includes all dev commands
- [ ] `doctor` annotation is `agent_allowed: true` with reason documenting `--fix` restriction
- [ ] Manifest `version` field incremented
- [ ] `manifest_test.go` count and allowlist assertions pass in both profiles
- [ ] New completeness test asserts no command is missing its annotation
- [ ] Both-profile gate passes