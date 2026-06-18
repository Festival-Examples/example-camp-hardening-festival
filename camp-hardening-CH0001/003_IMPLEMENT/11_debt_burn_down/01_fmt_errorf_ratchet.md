---
fest_type: task
fest_id: 01_fmt_errorf_ratchet.md
fest_name: fmt_errorf_ratchet
fest_parent: 11_debt_burn_down
fest_order: 1
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.787029-06:00
fest_updated: 2026-06-15T03:33:20.174756-06:00
fest_tracking: true
---


# Task: fmt.Errorf Lint Ratchet and Priority Migration

## Objective

Prevent new `fmt.Errorf` occurrences in production code (outside `tools/`) by adding a grep-based ratchet recipe mirroring the existing `lint-no-host-fs-tests` pattern, then migrate the two highest-propagation sites first: `internal/git/query.go` and `pkg/commitkit`.

## Requirements

- [ ] A `lint-no-fmt-errorf` just recipe (or equivalent) blocks any new `fmt.Errorf` in non-test production code outside `tools/` while allowing the pre-existing count via an allowlist file
- [ ] `internal/git/query.go:16` migrated from `fmt.Errorf` to `camperrors`; stderr routed (currently `cmd.Output()` discards it)
- [ ] All `pkg/commitkit` occurrences migrated to `camperrors` (9 non-test occurrences across `commitkit.go` and `autowrite.go`)
- [ ] Both-profile gate remains green after each commit
- [ ] The ratchet recipe is wired into `just lint`
- [ ] A test confirms that calling the ratchet recipe with a new `fmt.Errorf` in a production file outside the allowlist exits non-zero

## Implementation

### Background

`ERR-1` reports 327 `fmt.Errorf` occurrences in non-test production code at review time. The worktree at `bc2ad1f` shows the same counts: `cmd/` 111, `internal/` 195, `pkg/` 9, `tools/` 12. There is no `.golangci.yml` and the existing `lint` recipe runs `go vet` only. The project already has a working grep-based ratchet pattern at `justfile:71-96` for host-fs test violations (`lint-no-host-fs-tests`). This task mirrors that exact pattern for `fmt.Errorf`, which avoids introducing `golangci-lint` as a new dependency (consistent with C-7: external dependencies only when they provide 2-3x value).

**Why query.go first:** `internal/git/query.go:16` is the `Output` helper used across git plumbing. Every caller inherits the untyped error and the discarded stderr. Migrating it also surfaces the stderr problem.

**Why commitkit first:** `pkg/commitkit` is fest's public API. fest currently receives untyped errors. Migrating it makes the interface typed before sequence 12 polishes it.

### Step 1: Add the allowlist file

Create `.justfiles/fmt-errorf-allowlist.txt` at the worktree root. This file lists every existing non-test production file containing `fmt.Errorf` outside `tools/`. Generate it:

```bash
grep -rn "fmt.Errorf" cmd/ internal/ pkg/ \
  --include="*.go" | grep -v "_test.go" \
  | cut -d: -f1 | sort -u
```

Write each path on its own line, one per file. This is the burn-down list.

### Step 2: Add the ratchet recipe to `justfile`

Add the following recipe after `lint-no-host-fs-tests`. The structure matches the existing ratchet exactly:

```just
# Reject NEW fmt.Errorf occurrences in production code outside tools/.
# The allowlist captures pre-existing violators tracked under CH0001-11-01.
# Removing a file from the allowlist as you migrate it tightens the rule.
# The end state is an empty allowlist.
lint-no-fmt-errorf:
    #!/usr/bin/env sh
    set -eu
    allowlist=$(cat .justfiles/fmt-errorf-allowlist.txt 2>/dev/null || true)
    hits=$(find ./cmd ./internal ./pkg -name '*.go' \
        -not -name '*_test.go' -print0 2>/dev/null | \
        xargs -0 grep -l "fmt\.Errorf" 2>/dev/null || true)
    new_violators=""
    for hit in $hits; do
        case " $allowlist " in
            *" $hit "*) ;;
            *) new_violators="$new_violators $hit" ;;
        esac
    done
    if [ -n "$new_violators" ]; then
        echo "FAIL: NEW fmt.Errorf in production code (outside tools/):"
        for v in $new_violators; do echo "  $v"; done
        echo ""
        echo "Use camperrors.Wrap / camperrors.Wrapf instead."
        exit 1
    fi
    echo "lint-no-fmt-errorf: clean (no NEW violators)"
```

Wire it into the `lint` recipe alongside `lint-no-host-fs-tests`:

```just
lint:
    go vet ./...
    just lint-no-host-fs-tests
    just lint-no-fmt-errorf
```

After `BUILD_TAGS=dev just lint` verify it passes (all existing violators are in the allowlist).

### Step 3: Migrate `internal/git/query.go`

Current code at line 11-17:

```go
func Output(ctx context.Context, repoPath string, args ...string) (string, error) {
    fullArgs := append([]string{"-C", repoPath}, args...)
    cmd := exec.CommandContext(ctx, "git", fullArgs...)
    output, err := cmd.Output()
    if err != nil {
        return "", fmt.Errorf("git %s: %w", strings.Join(args, " "), err)
    }
    return strings.TrimSpace(string(output)), nil
}
```

Problems: `cmd.Output()` discards stderr; `fmt.Errorf` wraps without using `camperrors`; `strings.TrimSpace` on the full output strips the leading status column for porcelain output (GIT-4 is the full fix, but the stderr problem and the fmt.Errorf are fixed here).

Migrate to:

```go
import (
    camperrors "github.com/Obedience-Corp/camp/internal/errors"
)

func Output(ctx context.Context, repoPath string, args ...string) (string, error) {
    fullArgs := append([]string{"-C", repoPath}, args...)
    cmd := exec.CommandContext(ctx, "git", fullArgs...)
    var stderr bytes.Buffer
    cmd.Stderr = &stderr
    output, err := cmd.Output()
    if err != nil {
        return "", camperrors.Wrapf(err, "git %s: %s", strings.Join(args, " "), stderr.String())
    }
    return strings.TrimSpace(string(output)), nil
}
```

Add `"bytes"` to imports. Remove `"fmt"` if it is no longer used in the file. Remove `query.go` from the allowlist file.

Run `just build` and `BUILD_TAGS=dev just build` to confirm compilation.

### Step 4: Migrate `pkg/commitkit`

The nine occurrences span two files. Verify current state before editing:

```bash
grep -n "fmt.Errorf" pkg/commitkit/commitkit.go pkg/commitkit/autowrite.go
```

Expected locations in `commitkit.go`:
- `:95` commitkit: load campaign config at ... (root)
- `:106` commitkit: load campaign config at ... (campaignRoot)
- `:172` commitkit: rev-parse --short HEAD
- `:193` commitkit: stage submodule ...
- `:199` commitkit: check staged changes
- `:209` commitkit: commit submodule ref for ...

Expected locations in `autowrite.go`:
- `:51` commitkit: load campaign config at ...
- `:122` auto-write commit message command failed (with output)
- `:124` auto-write commit message command failed (without output)

For each occurrence replace `fmt.Errorf("...", ...)` with the appropriate `camperrors` call:

- Wrapping another error with context: `camperrors.Wrapf(err, "commitkit: ...")`
- Message with format verb but no wrapped error: `camperrors.Wrapf(camperrors.ErrGitFailed, "commitkit: ...")`

Examples:

```go
// Before:
return "", fmt.Errorf("commitkit: load campaign config at %s: %w", root, err)
// After:
return "", camperrors.Wrapf(err, "commitkit: load campaign config at %s", root)

// Before:
return fmt.Errorf("commitkit: stage submodule %s: %w", projectRelPath, err)
// After:
return camperrors.Wrapf(err, "commitkit: stage submodule %s", projectRelPath)
```

Add the camperrors import:

```go
import (
    camperrors "github.com/Obedience-Corp/camp/internal/errors"
)
```

Remove the `"fmt"` import from each file after migration (if no other usage remains).

Remove both `pkg/commitkit/commitkit.go` and `pkg/commitkit/autowrite.go` from the allowlist file.

Run:
```bash
just build
BUILD_TAGS=dev just build
go test ./pkg/commitkit/...
```

Existing commitkit tests must pass without change.

### Step 5: Ratchet test

Add a test to `justfile` test suite or as a standalone script that:
1. Creates a temporary file `tmptest_fmt.go` in `internal/` with a single `fmt.Errorf` call
2. Runs `just lint-no-fmt-errorf`
3. Asserts the exit code is non-zero
4. Removes the temporary file

This can be done as a shell assertion in a `test-ratchet` just recipe:

```just
test-ratchet:
    #!/usr/bin/env sh
    set -e
    tmpfile=./internal/tmptest_fmt_errorf_ratchet_test_file.go
    printf 'package internal\nimport "fmt"\nfunc _tmp() error { return fmt.Errorf("bad") }\n' > "$tmpfile"
    if just lint-no-fmt-errorf 2>&1 | grep -q "FAIL"; then
        rm -f "$tmpfile"
        echo "ratchet test: PASS (correctly rejected new fmt.Errorf)"
    else
        rm -f "$tmpfile"
        echo "ratchet test: FAIL (should have rejected new fmt.Errorf)"
        exit 1
    fi
    rm -f "$tmpfile"
```

### Step 6: Commit

Two commits:
1. `chore(lint): add fmt.Errorf ratchet recipe and allowlist` (justfile changes + allowlist file)
2. `fix(errors): migrate query.go and commitkit from fmt.Errorf to camperrors` (the two priority migration sites)

Run `just lint` in both profiles before each commit to confirm the ratchet stays green.

### Edge cases

- The `tools/` directory is explicitly exempt. The ratchet `find` command excludes it by only searching `./cmd ./internal ./pkg`.
- `autowrite.go:122` wraps an error where the message also contains a `%s` of the stderr output. Use `camperrors.Wrapf(err, "auto-write commit message command failed: %s", msg)`.
- Check that the `camperrors` package is already imported in files you are modifying; if not, add it. The import alias `camperrors` is the convention used by the codebase (`internal/errors` imported as `camperrors "github.com/Obedience-Corp/camp/internal/errors"`).

### Out of scope

- Migrating the remaining 300+ `fmt.Errorf` sites. Those are covered by the burn-down allowlist and migrate incrementally across subsequent sequences.
- Adding `golangci-lint` or `forbidigo`. The grep ratchet is the chosen approach per C-7.
- Changing `strings.TrimSpace` behavior in `query.go` (GIT-4 full fix is task 04).

## Done When

- [ ] `just lint` and `BUILD_TAGS=dev just lint` both exit 0 with the ratchet recipe green
- [ ] `just lint-no-fmt-errorf` exits non-zero when given a new violator (ratchet test passes)
- [ ] `grep -n "fmt.Errorf" internal/git/query.go pkg/commitkit/commitkit.go pkg/commitkit/autowrite.go` returns zero results
- [ ] `just build` and `BUILD_TAGS=dev just build` pass
- [ ] `go test ./internal/git/... ./pkg/commitkit/...` passes
- [ ] Both files removed from the allowlist; allowlist file committed with remaining pre-existing violators only