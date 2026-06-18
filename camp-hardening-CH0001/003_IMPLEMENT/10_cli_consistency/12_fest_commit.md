---
fest_type: gate
fest_id: 12_fest_commit.md
fest_name: Fest Commit Changes
fest_parent: 10_cli_consistency
fest_order: 12
fest_status: completed
fest_autonomy: high
fest_gate_id: fest-commit
fest_gate_type: commit
fest_managed: true
fest_created: 2026-06-12T00:56:20.225655-06:00
fest_updated: 2026-06-15T03:25:28.756714-06:00
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

## Result

Sequence 10 project changes were committed incrementally with `fest commit`:

- `0e22e83` JSON flag standardization
- `54e038b` exit code standardization
- `9369ab2` signal-aware CLI context
- `49e5b57` alias and usage fixes
- `c19d3d2` README install path and flow channel docs
- `df44209` shell wrapper transparency and nav checks
- `7c66e2c` init/cleanup failure surfacing and TTY guards
- `db157cf` pull/push flag handling

Final gate verification ran:

```bash
fest commit -m "chore: verify cli consistency sequence committed"
```

The command reported `nothing to commit, working tree clean`, confirming no
remaining project changes were left uncommitted.
