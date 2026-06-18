---
fest_type: task
fest_id: 01_fix_manifest_allowlist.md
fest_name: fix_manifest_allowlist
fest_parent: 01_gate_restoration
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.601985-06:00
fest_updated: 2026-06-12T13:20:00.291221-06:00
fest_tracking: true
---


# Task: Fix Manifest Allowlist for `workitem priority`

## Objective

Add `workitem priority` to the manifest test allowlist and bump the restricted-command count expectations so `TestManifestCommand` passes green in both build profiles.

## Requirements

- [ ] `"workitem priority": false` added to the `expectedCommands` workitem block in `TestManifestCommand_AllRestrictedCommandsPresent`, inside the `if workitemCommandRegistered()` guard.
- [ ] Workitem count increment changed from `wantCount += 9` to `wantCount += 10` in `TestManifestCommand_AllRestrictedCommandsPresent`.
- [ ] `agentAllowed["workitem priority"] = true` added to the workitem block in `TestManifestCommand_AllCommandsHaveAnnotations`, inside the `if workitemCommandRegistered()` guard.
- [ ] `go test -count=1 -short -run TestManifestCommand ./cmd/camp` passes with no failures in stable profile.
- [ ] `go test -tags dev -count=1 -short -run TestManifestCommand ./cmd/camp` passes with no failures in dev profile.
- [ ] No other changes to `manifest_test.go` or any other file. Out of scope: every other red item found in the baseline.

## Implementation

### Background

PR #318 merged `camp workitem priority` (`internal/commands/workitem/priority.go`) with `agent_allowed: "true"` set in its annotations (line 39). The command was registered but never added to the two manifest test maps that track the restricted-command surface. No gate ran at merge time (TEST-1 in the findings; BR-2 explains the structural cause: no CI exists).

The failing test output from the review session:

```
--- FAIL: TestManifestCommand_AllRestrictedCommandsPresent (manifest_test.go:162: expected exactly 27 restricted commands, got 28)
--- FAIL: TestManifestCommand_AllCommandsHaveAnnotations (manifest_test.go:222: command "workitem priority" is agent_allowed=true but not in allowlist)
```

Same two failures under `-tags dev` (43 vs 44). The dev failure is identical because `workitem priority` is registered in both profiles (no `//go:build dev` guard in its registration).

Why it matters: `just test unit` and `just build full` fail for everyone on this branch. Any later sequence that closes against a testing gate while this is unfixed will either inherit a red suite or be silently skipped. This task is the bootstrap precondition for the entire festival (D007).

### Verified file:line targets

File: `cmd/camp/manifest_test.go` in the worktree at `projects/worktrees/camp/camp-hardening` (commit `bc2ad1f`).

Review citations were "~:150 count block, ~:162 and ~:209/:222". Verified actual line numbers after reading the file at `bc2ad1f`:

- **Lines 127-137**: `if workitemCommandRegistered()` block in `TestManifestCommand_AllRestrictedCommandsPresent` that populates `expectedCommands`. The closing `}` is at line 137. Add the new entry before line 137.
- **Line 159**: `wantCount += 9` (the workitem addend). Change to `wantCount += 10`.
- **Line 161-163**: `if len(manifest.Commands) != wantCount { t.Errorf(...) }`. This is where the count failure fires. No edit needed here; the count fix at line 159 resolves it.
- **Lines 208-218**: `if workitemCommandRegistered()` block in `TestManifestCommand_AllCommandsHaveAnnotations` that populates `agentAllowed`. The closing `}` is at line 218. Add the new entry before line 218.
- **Line 221-222**: `if cmd.AgentAllowed && !agentAllowed[cmd.Path]` assertion. This is where the allowlist failure fires. No edit needed; the map addition at lines 208-218 resolves it.

No line drift found between the review citations and the worktree. The review cited "~:150 count block" as approximate; the wantCount block actually starts at line 151 (`wantCount := 18`). All other cited lines are exact.

### Step-by-step approach

**Step 1.** Open `cmd/camp/manifest_test.go` in the worktree. Read the file to confirm the lines match what is described above before editing.

**Step 2.** In `TestManifestCommand_AllRestrictedCommandsPresent`, inside the `if workitemCommandRegistered()` block (lines 127-137), add `"workitem priority"` as a new entry. The block currently ends:

```go
		expectedCommands["workitem commit"] = false
		expectedCommands["workitem commits"] = false
	}
```

Change to:

```go
		expectedCommands["workitem commit"] = false
		expectedCommands["workitem commits"] = false
		expectedCommands["workitem priority"] = false
	}
```

**Step 3.** On line 159, change the workitem count increment:

```go
	if workitemCommandRegistered() {
		wantCount += 9
	}
```

to:

```go
	if workitemCommandRegistered() {
		wantCount += 10
	}
```

**Step 4.** In `TestManifestCommand_AllCommandsHaveAnnotations`, inside the `if workitemCommandRegistered()` block (lines 208-218), add `"workitem priority"` to the allowlist. The block currently ends:

```go
		agentAllowed["workitem commit"] = true
		agentAllowed["workitem commits"] = true
	}
```

Change to:

```go
		agentAllowed["workitem commit"] = true
		agentAllowed["workitem commits"] = true
		agentAllowed["workitem priority"] = true
	}
```

**Step 5.** Save the file. Run the targeted test in stable profile to confirm green:

```bash
go test -count=1 -short -run TestManifestCommand ./cmd/camp
```

Expected output: all four `TestManifestCommand_*` subtests pass, no FAIL lines.

**Step 6.** Run the same test in dev profile:

```bash
go test -tags dev -count=1 -short -run TestManifestCommand ./cmd/camp
```

Expected output: same four subtests pass. The dev count goes from 43 to 44 (18 base + 3 flow + 13 quest + 10 workitem).

### Edge cases

The three `workitemCommandRegistered()` blocks in the test file are independent guards. All three are in the same file and the `workitem priority` entry touches only two of them (the `expectedCommands` map and the `agentAllowed` map). The third block, in `TestManifestCommand_InteractiveFlags` (lines 290-300), covers `nonInteractiveCommands` and does NOT need a new entry because `workitem priority` is not interactive. Do not add it there.

### Out of scope

Every other red item found during the baseline survey (task 02) belongs to its owning sequence. Do not touch any other file. Do not fix TEST-2, GIT-1, or any other finding in this task.

## Done When

- [ ] All requirements met
- [ ] `go test -count=1 -short -run TestManifestCommand ./cmd/camp` exits 0 with output showing all four subtests passing
- [ ] `go test -tags dev -count=1 -short -run TestManifestCommand ./cmd/camp` exits 0 with output showing all four subtests passing
- [ ] Diff of `cmd/camp/manifest_test.go` shows exactly three additions: the `expectedCommands` entry, the `wantCount` addend change, and the `agentAllowed` entry
- [ ] No other files modified