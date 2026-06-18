---
fest_type: task
fest_id: 06_quest_robustness.md
fest_name: quest_robustness
fest_parent: 11_debt_burn_down
fest_order: 6
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.788229-06:00
fest_updated: 2026-06-15T08:49:10.757046-06:00
fest_tracking: true
---


# Task: Quest Subsystem Robustness

## Objective

Harden the quest subsystem against corrupt data, concurrent creates, and channel-boundary leakage so that a single bad quest.yaml can never brick all quest commands and quest surfaces in stable builds are either fully functional or fully absent.

## Requirements

- [ ] `internal/quest/quest.go List` skips-and-warns on bad entries instead of hard-failing
- [ ] Slug reservation uses `os.Mkdir` (EEXIST = try-next-suffix) instead of `os.Stat` TOCTOU
- [ ] Quest ID uniqueness verified at create time
- [ ] BR-5 stable-surface decision: channel-aware scaffolding for `camp init` quest dirs and channel-aware `--quest` flag help text, OR explicit forward-compat documentation; recommendation below
- [ ] Atomic quest writes already done in sequence 06; this task references, does not redo them
- [ ] Both-profile gate green; both-profile tests pass

## Implementation

### Background

The quest subsystem has four inter-related findings:
- `List` at `internal/quest/quest.go:134` returns on the first `Load` error, bricking all quest commands (list, show, link, lifecycle, CAMP_QUEST resolution) when any single quest.yaml is corrupt
- `uniqueQuestDir` at `service.go:480-483` reserves by `os.Stat` (TOCTOU): two concurrent creates with the same name and timestamp see the directory as absent simultaneously and overwrite each other
- `GenerateID` at `slug.go:55-61` generates a random ID without checking for collisions with existing quests
- Stable binaries scaffold quest dungeon at `internal/scaffold/init.go:219` and the `--quest` flag is visible in stable `workitem create/adopt`, but `camp quest` does not exist in stable (BR-5 half-in/half-out)

Atomic quest writes (sequence 06 task 01) are a prerequisite for full correctness but this task focuses on the above four issues.

### Step 1: Fix `List` to skip-and-warn on bad entries

Location: `internal/quest/quest.go` around line 134. The current loop:

```go
q, err := Load(ctx, QuestPathForDir(filepath.Join(QuestsDir(campaignRoot), entry.Name())))
if err != nil {
    return nil, err  // hard fail
}
quests = append(quests, q)
```

Change to:

```go
q, err := Load(ctx, QuestPathForDir(filepath.Join(QuestsDir(campaignRoot), entry.Name())))
if err != nil {
    // Skip corrupt quests rather than bricking all quest commands.
    // The warning goes to stderr so callers can surface it.
    fmt.Fprintf(os.Stderr, "warning: skipping unreadable quest %q: %v\n", entry.Name(), err)
    continue
}
quests = append(quests, q)
```

Apply the same skip-and-warn pattern to the `includeDungeon` loop for status subdirs (the second loop also has a `return nil, err` on load failure; check the worktree and apply the same fix).

Test:
```go
func TestListSkipsCorruptQuest(t *testing.T) {
    // Create a campaign with two quests; write invalid YAML to one quest.yaml
    // Call List; expect exactly one quest returned, not an error
}
```

### Step 2: Fix slug reservation TOCTOU

Location: `internal/quest/service.go uniqueQuestDir` around `:480`. Current:

```go
dir := QuestDir(s.campaignRoot, base)
if _, err := os.Stat(dir); os.IsNotExist(err) {
    return dir, nil  // TOCTOU window here
}
```

Replace with `os.Mkdir`-based claim:

```go
func (s *Service) claimQuestDir(ctx context.Context, name string, now time.Time) (string, error) {
    base := GenerateDirectorySlug(name, now)
    for i := 0; i < 1000; i++ {
        if err := ctx.Err(); err != nil {
            return "", err
        }
        slug := base
        if i > 0 {
            slug = fmt.Sprintf("%s-%d", base, i+1)
        }
        dir := QuestDir(s.campaignRoot, slug)
        err := os.Mkdir(dir, 0755)
        if err == nil {
            return dir, nil // claimed atomically
        }
        if !os.IsExist(err) {
            return "", camperrors.Wrapf(err, "create quest dir %s", dir)
        }
        // EEXIST: try next suffix
    }
    return "", camperrors.Wrapf(camperrors.ErrInvalidInput,
        "could not allocate quest directory for %q after 1000 attempts", name)
}
```

Replace the call to `uniqueQuestDir` in `Create` and `CreateWithEditor` with `claimQuestDir`. The `os.Mkdir` is the actual creation so the caller must NOT call `os.MkdirAll` again after.

Test:
```go
func TestConcurrentSameNameQuestCreate(t *testing.T) {
    // Launch two goroutines simultaneously creating quests with the same name
    // and timestamp. Both should succeed with distinct directories.
    // (Use t.TempDir() for the campaign root; no real git needed)
}
```

### Step 3: Verify quest ID uniqueness at create

Location: `internal/quest/slug.go GenerateID` at `:55-61`. Currently generates a random suffix without checking existing quests.

Add a uniqueness check in `Service.Create`:

```go
// After generating id via GenerateID:
existing, _ := List(ctx, s.campaignRoot, ListOptions{IncludeDungeon: true})
for _, eq := range existing {
    if eq.ID == id {
        // Regenerate once; if still collides, return error
        id2, err := GenerateID(now)
        if err != nil {
            return nil, camperrors.Wrap(err, "generate quest id")
        }
        id = id2
        break
    }
}
```

This is a best-effort check (not a hard guarantee due to the TOCTOU between check and claim), but it catches the common same-second duplicate case. The `claimQuestDir` in step 2 is the true concurrency guard.

### Step 4: BR-5 channel-aware quest surfaces

The finding: stable binaries scaffold quest dungeon (`internal/scaffold/init.go:219`) and the `--quest` flag is registered in stable `workitem create/adopt`, but `camp quest` does not exist in stable.

**Recommendation: channel-aware approach** (same mechanism as D002 for flow): use the `Profile` constant introduced in sequence 09 task 08. The `//go:build dev` / `//go:build !dev` constant pair makes the Profile available at runtime.

Two changes:

**A. Channel-aware scaffolding in `internal/scaffold/init.go:219`:**
```go
// Only scaffold quest dungeon in dev-profile builds
if version.Profile == version.ProfileDev {
    questResult, err := quest.EnsureQuestDungeon(ctx, absDir)
    if err != nil {
        return nil, camperrors.Wrap(err, "failed to initialize quest dungeon")
    }
    // ... append results
}
```

**B. Channel-aware `--quest` flag help text:**
In `internal/commands/workitem/create.go:46` and `adopt.go:31`, change the flag description to be profile-conditional or simply suppress the flag registration in stable. The simpler option is a descriptive change that makes the forward-compat intent explicit:
```go
// Stable: register flag but with help text noting dev-only activation
cmd.Flags().StringVar(&questSelector, "quest", "",
    "quest ID to associate (requires dev-profile camp; forward-compatible flag)")
```

If the Profile constant is not yet available (sequence 09 hasn't run), use a build-tag check directly:
- Create `internal/commands/workitem/quest_flag_dev.go` (build tag `dev`) that calls `cmd.Flags().StringVar(&questSelector, "quest", "", "capture quest_id from this quest (defaults to CAMP_QUEST env var)")` 
- Create `internal/commands/workitem/quest_flag_stable.go` (build tag `!dev`) that registers the flag with the forward-compat description

Justify: this matches the same channel-aware pattern used for flow (D002) and avoids the half-in/half-out state for stable users who get quest scaffolding but no quest command.

Test:
```go
// In cmd/camp/release_profile_stable_test.go (or a new quest surface test):
func TestStableQuestScaffolding(t *testing.T) {
    // Run camp init in stable profile via the test binary
    // Verify .campaign/quests/ is NOT created
}
```

Note: if the Profile constant from sequence 09 is not yet available at this task's execution time, implement the build-tag approach (option B above) and note it for sequence 09 to clean up.

### Step 5: References to sequence 06 atomic writes

Sequence 06 task 01 converted `internal/quest/quest.go:271` from `os.WriteFile` to `fsutil.WriteFileAtomically`. Verify this is in place before closing this task:

```bash
grep -n "os.WriteFile\|WriteFileAtomically" internal/quest/quest.go
```

If the sequence 06 atomic write has not landed yet, add `quest.go:271` to this task's scope.

## Done When

- [ ] A corrupt `quest.yaml` does not cause `List` to return an error; the bad entry is skipped with a stderr warning
- [ ] `TestListSkipsCorruptQuest` passes
- [ ] `TestConcurrentSameNameQuestCreate` passes (two concurrent creates yield two distinct directories)
- [ ] Quest ID uniqueness check present in `Service.Create`
- [ ] Stable `camp init` does not scaffold `.campaign/quests/` (channel-aware build tag or Profile check in place)
- [ ] `grep -n "os.Stat(dir)" internal/quest/service.go` returns 0 results in the slug-reservation path (replaced by os.Mkdir)
- [ ] Both-profile gate green