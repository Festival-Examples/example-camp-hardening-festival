# Decisions Index

Registry of architecture decisions made during planning.

| ID | Decision | Status | Date |
|----|----------|--------|------|
| [D001](D001_n4_only_commit_semantics.md) | N-4 `--only` fix: temporary GIT_INDEX_FILE snapshot, staged-blob semantics for --staged | accepted | 2026-06-12 |
| [D002](D002_flow_channel.md) | R9 flow channel: channel-aware scaffolding and docs, no stable promotion | accepted | 2026-06-12 |
| [D003](D003_statusdir_primitive_timing.md) | R25 timing: surgical P0 migration fixes first, shared primitive in debt burn-down | accepted | 2026-06-12 |
| [D004](D004_schema_bumps.md) | JSON schema bumps: intents/flow-items/status-all/concepts/version new contracts; workitems v1alpha6; manifest version bump; app coordination workitem | accepted | 2026-06-12 |
| [D005](D005_prepush_hook.md) | Pre-push hook: .githooks plus just hooks install; gate-fast in hook, full just gate for releases and sequences | accepted | 2026-06-12 |
| [D006](D006_polish_scope.md) | POLISH is verification-only; all code sweeps live in IMPLEMENT | accepted | 2026-06-12 |
| [D007](D007_gate_restoration_bootstrap.md) | R1 gate restoration runs as a scope-capped bootstrap before the data-loss sequences; R13-R15 stay after them | accepted | 2026-06-12 |

## Status Values

- `proposed` — Under consideration
- `accepted` — Approved and final
- `superseded` — Replaced by a later decision

## Decision Template

Create `D###_title.md` files for each significant decision:

```markdown
# D001: [Decision Title]

**Status:** proposed | accepted | superseded
**Date:** YYYY-MM-DD

## Context

[Why is this decision needed?]

## Options

### Option A: [Name]
- **Pros:** [Benefits]
- **Cons:** [Drawbacks]

### Option B: [Name]
- **Pros:** [Benefits]
- **Cons:** [Drawbacks]

## Decision

[Which option was chosen and why]

## Consequences

[What changes or follow-up work results from this decision]
```
