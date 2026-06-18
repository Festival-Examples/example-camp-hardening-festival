---
fest_type: gate
fest_id: 08_fest_commit.md
fest_name: Fest Commit Changes
fest_parent: 12_test_hygiene_and_docs
fest_order: 8
fest_status: completed
fest_autonomy: high
fest_gate_id: fest-commit
fest_gate_type: commit
fest_managed: true
fest_created: 2026-06-12T00:56:20.230986-06:00
fest_updated: 2026-06-15T16:49:34.213073-06:00
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

## Committed Sequence Changes

- `ac80eb8` `[OBEY-CAMPAIGN-8deed8b4-FE-CH0001] fix: apply miscellaneous hardening polish`
- `b47810c` `[OBEY-CAMPAIGN-8deed8b4-FE-CH0001] chore(deps): pin guild-scaffold@v0.1.0 and obey-shared@v0.4.4`
- `0fe3d47` `[OBEY-CAMPAIGN-8deed8b4-FE-CH0001] test: align attachment pin integration expectation`
- `5db31ec` `[OBEY-CAMPAIGN-8deed8b4-FE-CH0001] fix(project): preserve link marker rollback`

## Verification

- Project branch clean and pushed at `5db31ec9753d1c44a3e6dd69229223405a16cfc4`.
- Commit identity scan over `origin/main..HEAD` found no author/committer mismatches; commits use `lancekrogers <lancekrogers@gmail.com>`.
- Latest gates: `GOFLAGS=-buildvcs=false go test ./internal/project`; `GOFLAGS=-buildvcs=false just test unit`; `GOFLAGS=-buildvcs=false just lint`; `git diff --check`.