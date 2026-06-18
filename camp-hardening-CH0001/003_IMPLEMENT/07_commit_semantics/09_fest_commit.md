---
fest_type: gate
fest_id: 09_fest_commit.md
fest_name: Fest Commit Changes
fest_parent: 07_commit_semantics
fest_order: 9
fest_status: completed
fest_autonomy: high
fest_gate_id: fest-commit
fest_gate_type: commit
fest_managed: true
fest_created: 2026-06-12T00:56:20.217031-06:00
fest_updated: 2026-06-14T23:51:54.963657-06:00
fest_tracking: true
fest_version: "1.0"
---


# Gate: Commit Sequence Changes

Commit all changes from this sequence using the `fest commit` command.

## Pre-Commit Checklist

- [ ] All tests pass
- [ ] Linting is clean
- [ ] No debug code or temporary files
- [ ] No secrets or credentials in staged changes

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

- [ ] Pre-commit checklist verified
- [ ] Commit created with `fest commit` (not `git commit`)
- [ ] Message describes what changed and why
- [ ] No prohibited content in commit message

## Completion Notes

- Sequence implementation commits were already created with `fest commit`:
  `b7ed55b`, `e2488b8`, `36981cf`, and `cefad5e`.
- Final gate command run: `fest commit -m "fix: finalize commit semantics hardening"`.
- Result: no additional project commit was created because the camp worktree was clean (`nothing to commit, working tree clean`).
- Pre-commit checklist verified by the sequence testing gate: stable/dev builds, stable/dev unit tests, lint, vet, and integration tests passed.