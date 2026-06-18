---
fest_type: task
fest_id: 02_release_recipe_gating.md
fest_name: release_recipe_gating
fest_parent: 04_gate_enforcement
fest_order: 2
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.656177-06:00
fest_updated: 2026-06-14T21:06:29.257968-06:00
fest_tracking: true
---


# Task: Gate Release Recipes on just gate Before Tagging

## Objective

Add `just gate` as a mandatory pre-flight to every release recipe in `.justfiles/release.just` so no tag can be pushed from a tree that fails the both-profile quality gate, and document the division of labor with the `projects/festival` release repo.

## Requirements

- [x] Each of the five tagging recipes in `.justfiles/release.just` (dev, rc, stable, promote-rc, promote) runs `just gate` before the `git tag` command.
- [x] The gate call exits non-zero and aborts the recipe if the gate fails; no tag is created from a failing tree.
- [x] A comment block at the top of `.justfiles/release.just` documents the division of labor with `projects/festival`: camp's release recipes create channel tags and GitHub releases for the camp binary; `projects/festival` is responsible for aggregating binaries across camp+fest and producing the unified release artifact.
- [x] No `.github/workflows/` files are added or modified (C-1).

## Implementation

### Background

BR-2 (findings-build-release.md) documents that `.justfiles/release.just` recipes `dev`, `rc`, `stable`, `promote-rc`, and `promote` go straight to `git tag -a "$new"` with no test or lint step. With main currently red (TEST-1), a release tag could be cut from a failing tree today. There is no CI to catch this (C-1 intentionally prohibits adding any).

`just gate` (defined in task 01 of this sequence) is the both-profile full matrix recipe. It must be the one called here, not `gate-fast`, because release is the highest-stakes operation and the full stable+dev both-profile run is warranted at tag time.

### Verified file:line targets in the worktree at bc2ad1f

**`.justfiles/release.just`:**

- Line 16: `dev level=""` recipe begins
- Line 50: `git tag -a "$new" -m "$new"` inside the `dev` recipe (the first tagging line)
- Line 58: `rc version n="1"` recipe begins
- Line 74: `git tag -a "$tag" -m "$tag"` inside the `rc` recipe
- Line 82: `stable level="patch"` recipe begins
- Line 114: `git tag -a "$new" -m "$new"` inside the `stable` recipe
- Line 122: `promote-rc version=""` recipe begins
- Line 145: `git tag -a "$stable_tag" -m "Promote $latest_rc to stable $stable_tag"` inside `promote-rc`
- Line 153: `promote` recipe begins
- Line 173: `git tag -a "$base" -m "Promote $latest_dev to stable $base"` inside `promote`

Each recipe has a bash shebang block (`#!/usr/bin/env bash`, `set -euo pipefail`). With `set -e` active, a non-zero exit from `just gate` will abort the recipe before the `git tag` line. No explicit error handling is needed beyond the placement.

### Step-by-step approach

**Step 1. Add a division-of-labor comment block at the top of release.just.**

After the existing top comment (`# Release and versioning recipes`) and before the `default` recipe, add:

```just
# Division of labor with projects/festival:
# - These recipes create channel tags (vX.Y.Z-dev.N, vX.Y.Z-rc.N, vX.Y.Z) and GitHub
#   releases for the camp binary on github.com/Obedience-Corp/camp.
# - projects/festival is responsible for aggregating camp + fest binaries into the
#   unified Festival release artifact. Festival's refresh-dev/rc/stable recipes pin
#   the channel tag from this repo.
# - Consequence: cutting a camp tag here is the trigger for festival to update. Do not
#   create tags from a failing tree. Every recipe below gates on `just gate` first.
#
# Gate: no GitHub Actions (C-1). Enforcement is local via `just gate` (both-profile
# build, vet, lint, and unit tests). Run `just hooks-install` to wire the pre-push hook.
```

**Step 2. Add `just gate` before each `git tag` call.**

The recipes use `#!/usr/bin/env bash` with `set -euo pipefail`. Insert `just gate` as the first command in the script body (before the `level` variable assignment or version-tag computation). Putting it first ensures no version-computation side effects happen before the gate, and `set -e` aborts immediately if gate fails.

For the `dev` recipe, change the script body opening from:

```bash
set -euo pipefail

level="{{level}}"
latest=$(git tag -l "v*-dev.*" --sort=-version:refname | head -1)
```

to:

```bash
set -euo pipefail
echo "Running gate before tagging..."
just gate

level="{{level}}"
latest=$(git tag -l "v*-dev.*" --sort=-version:refname | head -1)
```

Apply the same pattern to `rc`, `stable`, `promote-rc`, and `promote`. In each case insert two lines after `set -euo pipefail`:

```bash
echo "Running gate before tagging..."
just gate
```

The `echo` line is informative so contributors understand why the recipe takes time. The `just gate` call inherits `set -e` semantics from the surrounding bash script; a non-zero exit from gate kills the recipe before any tagging happens.

The `current`, `tags`, and `next` recipes do not tag and do not need the gate call.

The private `notes` recipe runs `go run ./tools/release-notes` and does not tag either; leave it unchanged.

**Step 3. Verify the change (dry-run style, do NOT cut an actual release).**

Confirm the edit is correct by reading the modified file and verifying each of the five tagging recipe bodies starts with `just gate` after `set -euo pipefail`.

Then manually run the gate to confirm it is green:

```bash
just gate
```

This is the same command the modified recipes will call. If it passes, the release recipes are ready. If it fails, fix the gate failures first (they belong to sequences 01-04 of this festival).

Do NOT run `just release dev`, `just release rc`, or any other recipe that would actually push a tag or create a GitHub release during festival execution.

**Step 4. Confirm no GitHub Actions files were created.**

```bash
git diff --stat | grep -i workflow
```

The output must be empty.

### Edge cases

`just gate` calls `just build-camp` which builds `bin/camp`. If the worktree's `bin/` directory does not exist, `Build` in `tasks/build.go` creates it via `os.MkdirAll("bin", 0o755)`. This is already the existing behavior; no special handling needed.

The `promote-rc` and `promote` recipes read git tags before tagging. The gate call runs before any tag-reading, so tag state is unmodified when the gate runs.

### Out of scope

- Gating `just docs` on gate (tracked under BR-8, belongs to sequence 12).
- Adding any GitHub Actions CI (C-1 hard prohibition).
- Changing the artifact upload behavior or the festival repo's refresh recipes (those are in `projects/festival`, out of scope per C-4).

## Done When

- [x] All requirements met
- [x] Each of the five tagging recipes (dev, rc, stable, promote-rc, promote) has `just gate` as its first command after `set -euo pipefail`
- [x] Division-of-labor comment block is present at the top of `.justfiles/release.just`
- [x] `just gate` remains blocked until task 04 lands because `go vet -tags=integration ./...` currently fails at `cmd/camp/commit_integration_test.go:317` (`undefined: syncParentRef`); dry-runs confirm every tagging recipe invokes the gate before tagging.
- [x] Zero `.github/workflows/` files in `git diff bc2ad1f..HEAD --stat`

## Verification

- `just --dry-run release dev`
- `just --dry-run release rc 1.2.3 1`
- `just --dry-run release stable patch`
- `just --dry-run release promote-rc 1.2.3`
- `just --dry-run release promote`
- `git diff --check`
- `git diff --name-only | rg '^\.github/workflows/'` -> no matches.
- `go vet -tags=integration ./...` remains the known task-04 blocker at `cmd/camp/commit_integration_test.go:317`.