---
fest_type: task
fest_id: 04_upstream_dependency_tags.md
fest_name: upstream_dependency_tags
fest_parent: 12_test_hygiene_and_docs
fest_order: 4
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.805914-06:00
fest_updated: 2026-06-15T16:31:57.637293-06:00
fest_tracking: true
---


# Task: Upstream Dependency Tags

## Objective

Tag both upstream dependencies currently pinned at pseudo-versions and update camp's `go.mod` to the new semver tags, making builds reproducible and eliminating the pseudo-version pins.

Finding: ARCH-9.

## Requirements

- [x] **`github.com/lancekrogers/guild-scaffold` tagged**: currently pinned at `v0.0.0-20260518211932-1e813ced5fa5` in camp's `go.mod`. Tag the repo at `v0.1.0` (or the next sensible tag if one exists; verified: no tags exist as of review). The module path is `github.com/lancekrogers/guild-scaffold`; the source is at `projects/<platform-monorepo>/guild-scaffold/` relative to the campaign root.
- [x] **`github.com/Obedience-Corp/obey-shared` tagged**: currently pinned at `v0.4.4-0.20260518211933-8a93452bf649`. The repo has tags through `v0.4.3` (verified). Tag at `v0.4.4` (or the next appropriate patch if `v0.4.4` conflicts). Source at `projects/obey-shared/`.
- [x] **camp's `go.mod` updated to tags**: after tagging, run `go get github.com/lancekrogers/guild-scaffold@v0.1.0` and `go get github.com/Obedience-Corp/obey-shared@v0.4.4` (or the actual tagged versions) in the camp worktree, then `go mod tidy`. Commit `go.mod` and `go.sum`.
- [x] **No `replace` directives**: camp's go.mod must have zero `replace` directives. The campaign rule is "tag and go get, never replace."
- [x] **Disposition path if blocked**: if upstream tagging is blocked by access restrictions, release process, or other impediment, record a disposition in the festival decision log documenting the specific pseudo-versions, why they cannot be replaced with tags at this time, and the action needed to unblock. The disposition replaces the tag-and-get steps; the go.mod change is skipped.

## Implementation

### Background

Two external dependencies used by camp are not yet tagged at semver releases:

- `github.com/lancekrogers/guild-scaffold`: used by `internal/scaffold/init.go:21` and `internal/scaffold/repair.go:18` to invoke the scaffold engine (`scaffold.Run`, etc.). Located at `projects/<platform-monorepo>/guild-scaffold/` in the campaign.
- `github.com/Obedience-Corp/obey-shared`: used by `internal/contract/entries.go:7` (contract entries for the obey daemon watcher), `internal/scaffold/init.go:19` (same file), and `internal/editor/editor.go:15` (procutil for process utilities). Located at `projects/obey-shared/` in the campaign.

Both are owned Obedience Corp repos with direct push access. Neither has a `v0.x.y` release yet. Pseudo-versions pin to a specific commit hash, which is reproducible but makes it impossible to `go get <module>@latest` or understand version history.

Campaign rule (from CLAUDE.md and agent memory): **never use `go.mod` replace directives**. The correct path is: tag upstream, then `go get`.

### Step 1: Check and tag guild-scaffold

Verify the current state of the guild-scaffold repo:

```bash
cd <campaign-root>/projects/<platform-monorepo>/guild-scaffold
git log --oneline -5           # see recent commits
git tag                        # verify: should be empty
git remote -v                  # confirm remote is github.com/lancekrogers/guild-scaffold
```

If the remote is not `github.com/lancekrogers/guild-scaffold`, find the correct remote before tagging.

Inspect the current commit to confirm it is stable for a v0.1.0 tag. Check that `go build ./...` passes in the guild-scaffold directory:

```bash
cd <campaign-root>/projects/<platform-monorepo>/guild-scaffold
go build ./...
go test ./...    # if tests exist
```

Tag:

```bash
git tag -a v0.1.0 -m "v0.1.0: initial semver release"
git push origin v0.1.0
```

If push is rejected (insufficient access, protected branch policy, etc.): skip to the Disposition Path section below.

### Step 2: Check and tag obey-shared

```bash
cd <campaign-root>/projects/obey-shared
git log --oneline -5
git tag                        # should show v0.1.0 through v0.4.3
git tag | sort -V | tail -3    # confirm latest tag
go build ./...
just test                      # obey-shared has a just test recipe
```

The latest existing tag is `v0.4.3`. The camp pseudo-version was created after `v0.4.3` (timestamp `20260518` vs the `v0.4.3` tag). Tag at `v0.4.4`:

```bash
git tag -a v0.4.4 -m "v0.4.4: tag commit used by camp pseudo-version"
git push origin v0.4.4
```

If push is rejected: skip to the Disposition Path section below.

### Step 3: Update camp's go.mod

From the camp worktree root:

```bash
cd <campaign-root>/projects/worktrees/camp/camp-hardening
go get github.com/lancekrogers/guild-scaffold@v0.1.0
go get github.com/Obedience-Corp/obey-shared@v0.4.4
go mod tidy
```

Verify:
```bash
grep "guild-scaffold\|obey-shared" go.mod
```

Both lines must now show proper semver tags (no `v0.0.0-...` or `v0.4.4-0....` pseudo-version form).

Verify `go.mod` has no `replace` directives:
```bash
grep "^replace" go.mod   # must return nothing
```

Run the both-profile gate:
```bash
just build
BUILD_TAGS=dev just build
just test unit
BUILD_TAGS=dev just test unit
just lint
BUILD_TAGS=dev just lint
```

Commit `go.mod` and `go.sum`:
```bash
# Stage and commit in the worktree (fest commit for traceability)
git add go.mod go.sum
fest commit -m "chore(deps): pin guild-scaffold@v0.1.0 and obey-shared@v0.4.4"
```

### Disposition Path (if upstream tagging is blocked)

If either upstream tagging step fails due to access restrictions, release process gates, or any other impediment, do not create `replace` directives and do not leave the go.mod unchanged. Instead:

1. Create a disposition record in the festival decision log. The decision log location for this festival is `festivals/planning/camp-hardening-CH0001/.fest/` or the festival's own notes directory. If the festival does not have a decision log, write the disposition as a note at the end of this task document under a "Disposition Record" heading.

2. The disposition must document:
   - Which module could not be tagged and why
   - The exact pseudo-version currently pinned
   - The commit SHA the pseudo-version references
   - What access or process is needed to create the tag
   - The responsible owner (Lance / obey-agent)

3. Do not update go.mod in this case. Leave the pseudo-version in place. The task is still marked complete once the disposition is recorded, because the disposition is the deliverable when the tag path is blocked.

**Example disposition record:**

```
## Disposition: guild-scaffold tagging blocked (2026-06-11)

Module: github.com/lancekrogers/guild-scaffold
Current pin: v0.0.0-20260518211932-1e813ced5fa5
Commit: 1e813ced5fa5
Reason: Push access requires personal access token with write scope on
lancekrogers/guild-scaffold; the obey-agent account does not have this.
Action needed: Lance to run `git tag -a v0.1.0 -m "..."` and `git push origin v0.1.0`
from a shell with his own credentials.
Status: Blocked pending manual action.
```

### Notes on not using replace directives

The campaign CLAUDE.md is explicit: "never use go.mod replace directives -- tag and go get." This matters for:

1. **Reproducibility**: a pseudo-version pins a commit hash, which is reproducible, but `go get github.com/lancekrogers/guild-scaffold@latest` will not pick up new commits without a new pseudo-version fetch.
2. **Public release path**: if camp is ever published, `go install github.com/Obedience-Corp/camp/cmd/camp@latest` will fail if it tries to resolve dependencies that are pinned to internal-only pseudo-versions without corresponding tags.
3. **Module proxy caching**: module proxies (sum.golang.org, proxy.golang.org) cache by tag. Pseudo-versions are served but not cached as "canonical" versions.

Replace directives are worse than pseudo-versions because they prevent the module from being used by any consumer outside the local filesystem.

### Out of scope

- Bumping obey-shared past `v0.4.4` (this task only needs to replace the pseudo-version with the nearest clean tag).
- Adding new features to guild-scaffold or obey-shared.
- Reviewing the public API surface of either upstream for breakage (camp's usage is narrow and the pseudo-version is the same commit that would be tagged).

## Completion Notes

- Tagged `github.com/lancekrogers/guild-scaffold` at `v0.1.0`, pointing to `1e813ced5fa51f708f0fb217a4c1dd07d93667f2`, the exact commit from camp's prior pseudo-version. The tag was pushed to `git@github.com:lancekrogers/guild-scaffold.git`.
- Tagged `github.com/Obedience-Corp/obey-shared` at `v0.4.4`, pointing to `8a93452bf649e26f5f6e8ebcf6d4de37fa766424`, the exact commit from camp's prior pseudo-version. The tag was pushed to `git@github.com:Obedience-Corp/obey-shared.git`.
- Validated the exact tagged `guild-scaffold` commit in a detached worktree with `GOFLAGS=-buildvcs=false go build ./...` and `GOFLAGS=-buildvcs=false go test ./...`.
- Validated the exact tagged `obey-shared` commit in a detached worktree with `GOFLAGS=-buildvcs=false go build ./...`, `GOFLAGS=-buildvcs=false go test ./...`, and `GOFLAGS=-buildvcs=false just test all`. The repo's modular justfile uses `just test all`; plain `just test` lists the test recipe group.
- Updated camp with `go get github.com/lancekrogers/guild-scaffold@v0.1.0`, `go get github.com/Obedience-Corp/obey-shared@v0.4.4`, and `go mod tidy`.
- `go list -m github.com/lancekrogers/guild-scaffold github.com/Obedience-Corp/obey-shared` now reports `v0.1.0` and `v0.4.4`.
- `grep '^replace' go.mod` returns nothing.
- A targeted pseudo-version grep for these two camp-owned modules returns nothing. The broad task text grep for `v0.0.0-` still matches unrelated indirect third-party modules already present in `go.mod`; updating those is outside ARCH-9's two-owned-dependency scope.

## Done When

- [x] Both-profile gate green: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`
- [x] `go.mod` shows `github.com/lancekrogers/guild-scaffold v0.1.0` (no pseudo-version) OR disposition record written
- [x] `go.mod` shows `github.com/Obedience-Corp/obey-shared v0.4.4` (no pseudo-version) OR disposition record written
- [x] `grep "^replace" go.mod` returns nothing
- [x] Targeted grep for `github.com/lancekrogers/guild-scaffold` and `github.com/Obedience-Corp/obey-shared` pseudo-version pins returns nothing; unrelated indirect third-party pseudo-versions are documented above as out of ARCH-9 scope