# D002: R9 Flow Channel Contradiction

**Status:** accepted
**Date:** 2026-06-12

## Context

BR-6: scaffolded skills (`campaign-workflows/SKILL.md.tmpl`), `dungeon/OBEY.md.tmpl`, README, and the `dungeon add` Long help instruct `camp flow ...` commands, but `flow` is dev-only (`internal/commands/release/register_dev.go`). Every campaign initialized with a stable binary ships agent skills whose commands return "unknown command", a first-run failure against the <5 minute launch goal. The contradiction must be DECIDED, not papered over.

## Options

### Option A: Promote `flow` to stable
- **Pros:** Scaffolded content becomes true everywhere; simplest template story; the scaffold clearly assumed it.
- **Cons:** `internal/commands/flow` (~1,100 lines) has ZERO tests (TEST-4), a dead `--json` flag (N-7), help text contradicting behavior (`flow migrate`), and an advertised-but-unimplemented history feature. Promoting an immature surface to stable during a hardening festival expands the stable contract exactly when the festival is shrinking risk.

### Option B: Channel-aware scaffolding and docs
- **Pros:** Stable users never see commands they do not have; keeps the dev surface free to mature; consistent with the intentional channel design (a context fact the review honored).
- **Cons:** Templates need a profile input (build-tag constant, same mechanism as R21's `Profile` field); README and `dungeon add` help need channel-conditional or channel-neutral wording; slightly more moving parts.

## Decision

**Option B: channel-aware scaffolding.** The scaffolder renders flow instructions only when the running binary is dev-profile (using the `Profile` constant introduced by R21/09.08); README documents `flow` as dev-channel; the `dungeon add` Long help drops or conditions the `camp flow init` hint; `docs/cli-reference` stays profile-pinned (BR-5's `docs` recipe `BUILD_TAGS=''` pin rides along).

Rationale: flow fails three hardening criteria at once (zero tests, broken JSON contract, contradictory help). Promoting it to stable would convert N-7 and the flow help findings from dev-channel debt into stable-contract bugs. festival-app is unaffected either way (it bundles dev-profile camp). Re-evaluating promotion AFTER N-7 and the flow help fixes land is a post-festival product call, not this festival's scope.

## Consequences

- Task 10.05 implements: README install fix (BR-7), channel-aware templates, README flow caveat, dungeon-add help fix; depends on the `Profile` constant from 09.08 (sequence 09 precedes 10).
- Task 09.02 (N-7) still fixes `flow items --json` because festival-app uses dev-profile flow.
- The stable-channel quest scaffolding half-in/half-out finding (BR-5) is handled the same way in 11.06: scaffold and flags become channel-aware rather than promoting quest.
