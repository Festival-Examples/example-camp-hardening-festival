---
fest_type: gate
fest_id: 09_fest_commit.md
fest_name: Fest Commit Changes
fest_parent: 04_gate_enforcement
fest_order: 9
fest_status: completed
fest_autonomy: high
fest_gate_id: fest-commit
fest_gate_type: commit
fest_managed: true
fest_created: 2026-06-12T00:56:20.210292-06:00
fest_updated: 2026-06-14T21:42:28.595782-06:00
fest_tracking: true
fest_version: "1.0"
---


# Gate: Commit Sequence Changes

Commit all changes from this sequence using the `fest commit` command.

## Pre-Commit Checklist

- [x] All tests pass
- [x] Linting is clean
- [x] No debug code or temporary files
- [x] No secrets or credentials in staged changes

## Commit Command

You **MUST** use `fest commit`, not `git commit`. The `fest commit` command tags
commits with task reference IDs for tracking and metrics.

```bash
fest commit -m "<type>: <summary>"
```

**CRITICAL:** Do NOT use `git commit`, `git add && git commit`, or any other git
commit workflow. Always use `fest commit` so task references are preserved.

## Commit Message Format

```
<type>: <concise summary of changes>

<what changed: list concrete modifications>

<why it changed: purpose and motivation>
```

**Types:** `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

The message should describe WHAT changed and WHY. Be specific about files,
functions, or features that were added, modified, or removed.

## Ethical Requirements

The following practices are **prohibited** in commit messages:

- NO "Co-authored-by" tags for AI assistants
- NO AI tool attribution or advertisements
- NO links to AI services or products

## Definition of Done

- [x] Pre-commit checklist verified
- [x] Commit created with `fest commit` (not `git commit`)
- [x] Message describes what changed and why
- [x] No prohibited content in commit message

## Sequence Commits

Project worktree status is clean. Sequence code changes were committed with
`fest commit`:

- `c30d111` `test: honor build tags in local gates`
- `08c2107` `chore: gate release tagging recipes`
- `3755abc` `chore: add local pre-push gate hook`
- `0e2c998` `test: migrate integration-tagged host tests`
- `a96051c` `test: reset current container state paths`
- `81e9866` `test: preserve ff-only divergent pull coverage`

Verification after the final review fix:

- `go test -v -tags=integration ./tests/integration -run '^TestPullAll_FfOnlyDivergentBranchesFails$' -count=1 -timeout 5m`
- `go vet -tags=integration ./...`
- `just test integration` (368/368)
- `git diff --check`