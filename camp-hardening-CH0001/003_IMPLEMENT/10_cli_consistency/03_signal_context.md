---
fest_type: task
fest_id: 03_signal_context.md
fest_name: signal_context
fest_parent: 10_cli_consistency
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.769504-06:00
fest_updated: 2026-06-15T02:27:22.41669-06:00
fest_tracking: true
---


# Task: Signal-Aware Root Context

## Objective

Wire `signal.NotifyContext` in `main.go` so Ctrl-C and SIGTERM actually cancel in-flight operations, thread the resulting context through cobra via `ExecuteContext`, delete the nil-ctx fallback guards that were masking the gap, and replace the cancellation-ignoring `time.Sleep` in the git retry loop with a `select` that respects context cancellation.

## Requirements

- [ ] `cmd/camp/main.go`: construct `ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)` before calling `Execute(ctx)`; call `defer stop()`.
- [ ] `cmd/camp/root.go`: `Execute()` function signature changes to `Execute(ctx context.Context) error`; internal call changes from `rootCmd.Execute()` to `rootCmd.ExecuteContext(ctx)`.
- [ ] Nil-ctx fallbacks deleted from: `cmd/camp/commit.go:74-75,93-94`, `cmd/camp/sync.go:92-94`, `cmd/camp/doctor.go:85-87`, `cmd/camp/status_all.go:84-86`, `cmd/camp/stage.go:61-63`, `cmd/camp/worktrees/list.go:77-78`, `cmd/camp/worktrees/commit.go:62-63`, `cmd/camp/worktrees/create.go:65-66`, `cmd/camp/worktrees/clean.go:73-74`, `cmd/camp/worktrees/info.go:74-75`. Under `ExecuteContext`, cobra guarantees a non-nil context from `cmd.Context()`; the fallbacks are dead code.
- [ ] `internal/git/retry.go:159`: `time.Sleep(backoff)` replaced with a `select` statement that returns `ctx.Err()` on cancellation and sleeps for `backoff` otherwise.
- [ ] ERR-4 helper sites threaded with root context where the function is reachable from a cobra command (and therefore has access to `cmd.Context()`): `cmd/camp/root.go:119` (`loadShortcutsForExpansion`), `cmd/camp/navigation/go.go:453`, `cmd/camp/run.go:199`, `cmd/camp/plugin.go:53`, `cmd/camp/pins.go:210`. Pre-Execute argv rewriting sites (`loadShortcutsForExpansion` runs before cobra dispatch) legitimately use `context.Background()` because no signal context exists yet; document which helpers fall into this category and leave them with a comment explaining why they do not receive the root ctx.
- [ ] Both-profile gate passes after this task.

## Implementation

### Background

`cmd/camp/main.go:12` calls `Execute()` (no context). `cmd/camp/root.go:83` calls `rootCmd.Execute()` (the non-context variant). As a result, `cmd.Context()` always returns `context.Background()` with no cancellation signal wired. The codebase threads context excellently through I/O (every git exec uses `exec.CommandContext`), but nothing ever cancels that context, so a Ctrl-C only kills the process with default signal semantics rather than allowing lock-retry loops, multi-repo pulls, and container operations to clean up.

The nil-ctx guards (`if ctx == nil { ctx = context.Background() }`) in the cmd files were added defensively because the context was never set. Under `ExecuteContext`, cobra sets a non-nil context on every command before calling RunE; the guards become unreachable code.

The retry loop sleep at `internal/git/retry.go:159` is a clean, isolated fix: replace `time.Sleep(backoff)` with a `select` and it becomes cancellation-aware without any structural change.

Verify all cited line numbers against the worktree at `bc2ad1f` before editing.

### Verified file:line targets (as of review citations; verify before editing)

- `cmd/camp/main.go:12` - the `Execute()` call
- `cmd/camp/root.go:83` - `rootCmd.Execute()`
- `cmd/camp/commit.go:74-75,93-94` - nil-ctx fallbacks
- `cmd/camp/sync.go:92-94` - nil-ctx fallback
- `cmd/camp/doctor.go:85-87` - nil-ctx fallback
- `cmd/camp/status_all.go:84-86` - nil-ctx fallback
- `cmd/camp/stage.go:61-63` - nil-ctx fallback
- `cmd/camp/worktrees/*.go` - nil-ctx fallbacks across five files
- `internal/git/retry.go:159` - `time.Sleep(backoff)`
- `cmd/camp/root.go:119` - `loadShortcutsForExpansion` using `context.Background()`
- `cmd/camp/navigation/go.go:453` - fresh `context.Background()`
- `cmd/camp/run.go:199` - fresh `context.Background()`
- `cmd/camp/plugin.go:53` - fresh `context.Background()`
- `cmd/camp/pins.go:210` - fresh `context.Background()` with timeout

### Step-by-step approach

**Step 1: Wire signal context in `main.go`.**

Read `cmd/camp/main.go`. Add the following to the top of `main()`, before the `Execute()` call:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

if err := Execute(ctx); err != nil {
    // existing error handling...
}
```

Add imports for `"os/signal"` and `"syscall"` if not already present. The `context` import is likely already there. Remove the old `Execute()` call.

**Step 2: Update `Execute` to accept context.**

Read `cmd/camp/root.go`. Change the function signature from `func Execute() error` to `func Execute(ctx context.Context) error`. Change the internal call from `rootCmd.Execute()` to `rootCmd.ExecuteContext(ctx)`:

```go
func Execute(ctx context.Context) error {
    expandShortcuts()
    os.Args = normalizeAutoWriteAlias(os.Args)
    os.Args = normalizeOptionalValueFlagArgs(os.Args)

    if err := dispatchPlugin(); err != nil {
        if errors.Is(err, errPluginHandled) {
            return nil
        }
        return err
    }

    return rootCmd.ExecuteContext(ctx)
}
```

**Step 3: Delete nil-ctx fallbacks.**

For each listed file, read the RunE function and find the nil-ctx pattern:

```go
ctx := cmd.Context()
if ctx == nil {
    ctx = context.Background()
}
```

Delete the `if ctx == nil` block, leaving only `ctx := cmd.Context()`. Under `ExecuteContext`, `cmd.Context()` is always the context passed to `ExecuteContext`; the guard is dead. Verify each file individually before editing (lines may have drifted).

**Step 4: Fix the retry loop sleep.**

Read `internal/git/retry.go` near line 159. The current code:

```go
time.Sleep(backoff)
```

Replace with:

```go
select {
case <-ctx.Done():
    return ctx.Err()
case <-time.After(backoff):
}
```

The `ctx` variable is already in scope in the retry function (it is passed as a parameter). Verify the function signature takes `ctx context.Context` as the first argument before making this change.

**Step 5: Thread root context into reachable ERR-4 sites.**

Read each of the four reachable helper sites and determine whether they can receive a context from their caller:

- `cmd/camp/navigation/go.go:453`: if this is inside a RunE function that has `cmd.Context()`, pass `cmd.Context()` down instead of creating `context.Background()`.
- `cmd/camp/run.go:199`: same approach - check if `isProjectCtx` is called from a RunE that has a context.
- `cmd/camp/plugin.go:53`: check `dispatchPlugin`. It runs before cobra parses the command, so it does not have `cmd.Context()`. If it runs before `Execute(ctx)` (it does - it is called inside `Execute`), pass the `ctx` parameter down instead of creating a fresh one. Change `dispatchPlugin()` to `dispatchPlugin(ctx)` and update the function signature.
- `cmd/camp/pins.go:210`: if inside RunE, use `cmd.Context()`.

For `cmd/camp/root.go:119` (`loadShortcutsForExpansion`): this runs inside `expandShortcuts()` which runs before cobra dispatch and before `Execute(ctx)` receives its ctx. It legitimately uses `context.Background()`. Add a comment:

```go
// Pre-cobra argv rewriting: no signal context available yet.
ctx := context.Background()
```

Do not change its behavior; only document the legitimate exception.

**Step 6: Write tests.**

Write a test in `cmd/camp/root_test.go` (host `t.TempDir()` convention):

1. `TestExecuteContext_CancelledContext`: pass a pre-cancelled context to `Execute(ctx)`; verify it returns an error (the command should abort quickly without writing anything).

2. `TestRetryLoop_ContextCancellation` in `internal/git/retry_test.go`: create a context, cancel it after a short delay, start the retry loop with a long max backoff, verify it returns `ctx.Err()` promptly rather than sleeping through the full backoff.

The retry test can mock the underlying git lock operation with a function that always reports "locked" (so the loop always retries) and then confirm the loop exits via context rather than exhausting retries.

### Edge cases

**`dispatchPlugin` receives ctx before cobra runs:** The plugin dispatch runs before the full cobra parse, so passing the signal ctx into it is safe. The plugin child process will inherit the signal anyway since it is launched via `exec.CommandContext`, but threading the ctx ensures clean cancellation of the `exec.Command` before the child exits on its own.

**`loadShortcutsForExpansion` pre-cobra:** This function is intentionally excluded from receiving the signal context because it runs as part of pre-cobra argv rewriting. The comment documents this.

**Nil-ctx fallbacks in worktrees package:** All five worktrees files have the same two-line pattern. Delete all five. The pattern is in each file's RunE function at the top; none of them are called before `Execute`.

### Out of scope

- ERR-4 sites inside `internal/complete/complete.go:164` - this is called from the completion subsystem which is not a standard RunE; leave with a comment noting it is excluded.
- The ERR-4 finding describes `cmd/camp/root.go:119` as "pre-Execute argv rewriting"; this task documents the exception but does not change the code there.

## Done When

- [ ] All requirements met
- [ ] `cmd/camp/main.go` uses `signal.NotifyContext` and passes context to `Execute(ctx)`
- [ ] `cmd/camp/root.go:Execute` calls `rootCmd.ExecuteContext(ctx)`
- [ ] `grep -n "ctx == nil" cmd/camp/commit.go cmd/camp/sync.go cmd/camp/doctor.go cmd/camp/status_all.go cmd/camp/stage.go cmd/camp/worktrees/*.go` returns no results
- [ ] `internal/git/retry.go` backoff sleep uses a `select` with `ctx.Done()` branch
- [ ] `TestRetryLoop_ContextCancellation` test passes and completes in well under `maxBackoff` time when ctx is cancelled
- [ ] `TestExecuteContext_CancelledContext` test passes in both profiles
- [ ] Both-profile gate passes