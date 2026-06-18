---
fest_type: task
fest_id: 03_prepush_hook.md
fest_name: prepush_hook
fest_parent: 04_gate_enforcement
fest_order: 3
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.656553-06:00
fest_updated: 2026-06-14T21:09:47.008357-06:00
fest_tracking: true
---


# Task: Commit .githooks/pre-push Hook and just hooks-install Recipe

## Objective

Create a committed `.githooks/pre-push` hook that runs `just gate-fast` (or `just gate` when `CAMP_GATE_FULL=1`) before every push, add the `just hooks-install` recipe that activates it, and document the setup requirement in the contributor docs; no `.github/workflows/` files anywhere.

## Requirements

- [x] `.githooks/pre-push` is committed to the repo, is executable (`chmod +x`), runs `just gate-fast` by default, upgrades to `just gate` when `CAMP_GATE_FULL=1`, prints a clear failure banner naming the reproduce command, passes stdin through per the githooks(5) pre-push contract, and does not assume a TTY.
- [x] `just hooks-install` recipe in the root justfile sets `git config core.hooksPath .githooks` idempotently.
- [x] Contributor documentation notes the `just hooks-install` setup step (README or `docs/`, following repo conventions).
- [x] The hook works correctly from worktree checkouts and submodule contexts (the `core.hooksPath` config is per-repo; document the behavior).
- [x] Zero `.github/workflows/` files added or touched (C-1 hard constraint).

## Implementation

### Background

D005 specifies Option C: the hook runs `just gate-fast` (a bounded subset) with `CAMP_GATE_FULL=1` upgrading to `just gate`. The hook file is a two-liner delegating to versioned justfile recipes; policy lives in the justfile, not in the hook script.

The githooks(5) pre-push contract requires the hook to read and pass through stdin. A push to a remote sends lines on stdin in the format `<local ref> <local sha> <remote ref> <remote sha>`. If the hook ignores stdin and the network layer blocks waiting for the other end to read it, deadlocks can occur. The hook must consume stdin even if it does not use it.

The hook must not assume a TTY because git push is commonly run from non-interactive contexts (scripts, CI-equivalent local automation). `just gate-fast` does not require a TTY; the buildutil UI renders fine on non-TTY output.

### Step-by-step approach

**Step 1. Create the .githooks/ directory and hook file.**

From the worktree root:

```bash
mkdir -p .githooks
```

Create `.githooks/pre-push` with this exact content:

```sh
#!/usr/bin/env sh
# pre-push hook: run the local gate before pushing.
# Delegates to just gate-fast (default) or just gate (when CAMP_GATE_FULL=1).
# githooks(5): stdin carries <local-ref> <local-sha> <remote-ref> <remote-sha> per pushed ref.
# Must consume stdin even if unused to satisfy the protocol.

# Consume stdin per githooks(5) pre-push contract.
cat > /dev/null

if [ "${CAMP_GATE_FULL:-0}" = "1" ]; then
    gate_cmd="just gate"
else
    gate_cmd="just gate-fast"
fi

if ! $gate_cmd; then
    echo ""
    echo "========================================================"
    echo "  PRE-PUSH GATE FAILED"
    echo "  Reproduce locally: $gate_cmd"
    echo "  Skip (discouraged): git push --no-verify"
    echo "========================================================"
    exit 1
fi
```

Then make it executable:

```bash
chmod +x .githooks/pre-push
```

**Step 2. Verify the hook script is correct.**

Check the content:

```bash
cat .githooks/pre-push
```

Confirm:
- The shebang is `#!/usr/bin/env sh` (not bash; POSIX sh is sufficient and more portable).
- `cat > /dev/null` consumes stdin before running the gate, satisfying the githooks(5) pre-push protocol.
- The `CAMP_GATE_FULL` check uses `[ "${CAMP_GATE_FULL:-0}" = "1" ]` so the default is 0 and the var does not need to be exported.
- The failure banner names the reproduce command and mentions `--no-verify` as the discouraged bypass.
- `exit 1` is explicit on failure.

**Step 3. Add the hooks-install recipe to the root justfile.**

Task 01 of this sequence added `hooks-install` to the justfile as part of the gate recipe block. Verify it is present:

```bash
grep -n 'hooks-install' justfile
```

Expected: the recipe sets `git config core.hooksPath .githooks` and prints a confirmation. If task 01 has not landed yet, add it now following the same style (see task 01 step 3 for the full recipe body).

**Step 4. Document the setup in contributor docs.**

Find the contributor setup section. The camp repo has `README.md` with a development/contributing section. Locate it:

```bash
grep -n 'install-tools\|Development\|Contributing\|Setup' README.md | head -20
```

Add a note after the `just install-tools` step (or wherever development setup is described):

```markdown
### Install the pre-push gate hook

Run once after cloning or checking out this repo to activate the pre-push quality gate:

```sh
just hooks-install
```

This sets `core.hooksPath` to `.githooks` so git runs `.githooks/pre-push` before every push. The hook runs `just gate-fast` (both-profile builds, vet, lint, and dev unit tests). To run the full gate instead, set `CAMP_GATE_FULL=1` before pushing.
```

Match the existing README style for code blocks and prose (no emdashes, no emoji unless already used in that section).

**Step 5. Understand the worktree and submodule behavior.**

`git config core.hooksPath` is a per-repository config entry stored in `.git/config`. A `git worktree add` for the camp repo shares the same `.git/` directory as the parent checkout (worktrees use `.git/worktrees/<name>/` but read the parent's config). Therefore:

- Running `just hooks-install` in the camp repo main checkout also activates the hook for all worktrees of the same camp repo, including `projects/worktrees/camp/camp-hardening`.
- Submodule checkouts have their own `.git/config` (stored at `.git/modules/<name>/config` in the parent, symlinked from the submodule's `.git`). The hook is NOT automatically active in a submodule checkout; contributors working inside a submodule-registered camp copy must run `just hooks-install` in that checkout directory too.

Document this behavior in a comment inside the `hooks-install` recipe:

```just
# Set git core.hooksPath to .githooks so the pre-push gate fires automatically.
# Idempotent: safe to run multiple times.
# Note: worktrees of this repo share this config automatically.
# Submodule-registered copies have separate .git/config and need their own run.
hooks-install:
    git config core.hooksPath .githooks
    @echo "Hooks installed: .githooks is now the active hooks directory."
    @echo "To verify: git config --get core.hooksPath"
```

**Step 6. Scripted verification: scratch push showing the hook fires.**

Set up a local bare remote and verify the hook blocks on a deliberate failure:

```bash
# From the worktree root
cd /tmp
git clone --bare /path/to/worktree test-remote.git
cd /path/to/worktree

# Verify hooks-install activates the hook
just hooks-install
git config --get core.hooksPath   # should print: .githooks

# Deliberately make gate-fast fail by introducing a vet error in a scratch file
echo 'package main; func _() { var x int; _ = x; undefinedFunc() }' > /tmp/scratch_vet_test.go
cp /tmp/scratch_vet_test.go cmd/camp/scratch_vet_test.go

# Attempt a push (use a dummy remote that does not need network)
git remote add test-scratch /tmp/test-remote.git
git push test-scratch HEAD:refs/heads/test-hook-check 2>&1 || true
# Expected: pre-push gate fires, go vet detects the error, banner prints, push aborted

# Clean up
rm cmd/camp/scratch_vet_test.go
git remote remove test-scratch
```

The output must contain the `PRE-PUSH GATE FAILED` banner. The push must be aborted (non-zero exit). Record the actual output as verification evidence in the commit message or a task note.

**Step 7. Confirm no GitHub Actions files were created.**

```bash
ls .github/workflows/ 2>/dev/null && echo "ERROR: workflows directory exists" || echo "OK: no workflows"
git diff --stat | grep -i workflow
```

Both must show no `.github/workflows/` files.

### Edge cases

**TTY assumption.** `just gate-fast` calls `go run ./internal/buildutil build` and `go run ./internal/buildutil test`, both of which use the `buildutil/ui` package. The UI package writes to stdout/stderr using `fmt.Printf` and does not require a TTY (no `isatty` gating on the progress output). Output may be non-pretty in non-TTY contexts but the exit code behavior is correct.

**stdin consumption before gate.** Placing `cat > /dev/null` before the gate call means stdin is fully consumed before the gate runs. This is correct: the gate does not need the push ref list, and consuming stdin early prevents any potential blocking on the stdin pipe while the gate runs its child processes.

**Hook from an unrelated working directory.** When git runs the pre-push hook, the working directory is the repository root (or the worktree root for a worktree checkout). `just gate-fast` uses `[no-cd]` recipes that run relative to the `justfile` location. Since the justfile is at the repo root and git sets `cwd` to the repo root before running hooks, `just gate-fast` resolves correctly.

**`--no-verify` bypass.** The banner explicitly names the bypass so contributors know it exists. It is not blocked. The defense against bypass culture is that `just gate` is also required by the release recipes; bypassing the hook still requires passing gate at release time.

### Out of scope

- Any `.github/workflows/` files (C-1 hard prohibition; GitHub Actions is mentioned only as a prohibition).
- A `just doctor`-style check for whether `core.hooksPath` is set (tracked as an optional follow-up; not required for this task).
- The double-Reset in container tests (sequence 12 per C-4).

## Done When

- [x] All requirements met
- [x] `.githooks/pre-push` is committed and executable
- [x] `just hooks-install` is present and idempotent
- [x] Contributor docs note the `just hooks-install` setup step
- [x] Scripted scratch-push verification shows the hook firing and printing the failure banner on a deliberate gate failure
- [x] `git config --get core.hooksPath` prints `.githooks` after running `just hooks-install`
- [x] Zero `.github/workflows/` files in `git diff bc2ad1f..HEAD --stat`

## Verification

- `chmod +x .githooks/pre-push`
- `ls -l .githooks/pre-push` -> executable bit set.
- `sh -n .githooks/pre-push`
- `just --dry-run hooks-install`
- `just hooks-install`
- `git config --get core.hooksPath` -> `.githooks`
- Scratch push verification:
  - Added temporary `cmd/camp/scratch_vet_test.go` with a deliberate undefined function.
  - Added local bare remote `/tmp/camp-hardening-prepush-test.git`.
  - Ran `git push test-scratch HEAD:refs/heads/test-hook-check`.
  - Push exited non-zero and printed `PRE-PUSH GATE FAILED`, `Reproduce locally: just gate-fast`, and `failed to push some refs`.
  - Removed the scratch file and test remote afterward.
- `git diff --check`
- `git diff --name-only | rg '^\.github/workflows/'` -> no matches.