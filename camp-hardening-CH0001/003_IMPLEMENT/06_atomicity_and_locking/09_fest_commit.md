---
fest_autonomy: high
fest_created: 2026-06-12T00:56:20.214391-06:00
fest_gate_id: fest-commit
fest_gate_type: commit
fest_id: 09_fest_commit.md
fest_managed: true
fest_name: Fest Commit Changes
fest_order: 9
fest_parent: 06_atomicity_and_locking
fest_status: completed
fest_tracking: true
fest_type: gate
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

## Completion Notes

- Project worktree is clean after sequence commits.
- Sequence project commits were created with `fest commit`: `8177bb1`, `b26a20c`, `ffedd1b`, `2759cc6`, `5197dfe`, `975f49f`, `19d085f`, `1ec923f`, and `43c4acc`.
- Latest verification after the review iteration: `just build stable`, `just build dev`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, and `just vet` passed.
- Full integration verification for the sequence passed on rerun: `just test integration` reported 368/368 tests passed.
