# D005: Pre-push Hook Design (Local Gate Enforcement)

**Status:** accepted
**Date:** 2026-06-12

## Context

GitHub Actions is intentionally paused (C-1); enforcement is just recipes plus a local pre-push hook. The hook must be strong enough to stop the TEST-1 class (merged red gate) and fast enough that nobody bypasses it with `--no-verify` out of frustration.

## Options

### Option A: Full matrix in the hook
Hook runs build, full unit tests, and lint in both profiles (the complete gate). Strongest, but the camp unit suite is large (~2,678 host test functions); doubling it per push risks multi-minute pushes and bypass culture.

### Option B: Fast hook plus full `just gate` recipe
Hook runs a bounded subset (build both profiles, vet both tag sets including `-tags=integration`, lint, and the unit suite in the CURRENT profile); a `just gate` recipe runs the complete both-profile matrix and is the documented pre-merge/pre-release requirement (release recipes call it per BR-2).

### Option C: Hook delegates entirely to one recipe with an env opt-down
Hook runs `just gate-fast` (defined as B's subset) and honors `CAMP_GATE_FULL=1` to run `just gate`; everything lives in justfiles so the hook file is a two-liner and the policy is versioned with the repo.

## Decision

**Option C.** Concretely:

- `.githooks/pre-push` (committed): exec `just gate-fast` (or `just gate` when `CAMP_GATE_FULL=1`), with a clear failure banner naming the recipe to reproduce.
- `just hooks install`: sets `git config core.hooksPath .githooks` (idempotent); `camp`'s contributor docs and the release recipes mention it; `just doctor`-style check optional.
- `just gate-fast`: `build` stable + `build` dev, `go vet ./...` + `go vet -tags dev ./...` + `go vet -tags=integration ./...`, lint in BOTH profiles (`just lint` and `BUILD_TAGS=dev just lint`), unit tests in the dev profile (the profile festival-app ships, and the one historically never run; stable-only regressions are caught by `just gate`).
- `just gate`: gate-fast plus unit tests in BOTH profiles, so the full C-2 matrix (build, test, lint in stable and dev) runs in one recipe; required by release recipes before tagging (BR-2) and by the festival's own per-sequence quality gates.
- No `.github/workflows/` files are created or touched.

Rationale: the hook's job is stopping the silent-red-gate failure mode locally at push time; the full matrix belongs in the recipes that gate releases and festival sequences, where the time cost is expected.

## Consequences

- Task 04.01 implements BUILD_TAGS-honoring tests and the `gate`/`gate-fast` recipes; 04.02 wires release recipes to `just gate`; 04.03 adds the hook plus `just hooks install` and documents it.
- The festival's per-sequence verification plans reference `just gate` as the closing command.
- POLISH verifies `git config core.hooksPath` guidance exists and that no GitHub Actions files appear in the diff.
