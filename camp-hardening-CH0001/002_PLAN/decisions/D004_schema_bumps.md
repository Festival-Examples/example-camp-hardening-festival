# D004: JSON Schema Bumps and festival-app Coordination

**Status:** accepted
**Date:** 2026-06-12

## Context

Several contract fixes change JSON shapes festival-app parses. Per C-10, shape changes ship as versioned bumps with `docs/json-contracts.md` rows, never silent drift. The festival must enumerate which surfaces bump, which are new contracts, and how the app is notified.

## Decision

### New versioned contracts (no existing version to bump)

| Surface | Action | Task |
|---|---|---|
| intent list/find/count/show | New `intents/v1alpha1` envelope (schema_version, generated_at, campaign_root, items[]), `--json` flag with jsoncontract error envelope; `--format json` kept as a deprecated alias emitting the SAME new payload | 09.04 |
| flow items | New `workflow-items/v1alpha1` (or fold into the existing `workflow/v1` family if shapes align) via jsoncontract | 09.02 |
| status all | Version the existing snake_case payload (`status-all/v1alpha1`); empty case emits `{"schema_version":...,"timestamp":...,"repos":[]}` | 10.01 |
| concepts | New `concepts/v1alpha1` `--json` | 09.09 |
| version | Add `schema_version` plus `profile`; keep existing camelCase keys AND add snake_case duplicates documented as canonical (unreleased publicly, so a clean snake_case break is acceptable if simpler; implementer picks and documents) | 09.08 |

### Bumps to existing contracts

| Surface | From | To | Changes | Task |
|---|---|---|---|---|
| workitem list family | workitems/v1alpha5 | workitems/v1alpha6 | per-type stage vocabulary published; explicit no-stage representation; ritual/chains festivals included; drop omitempty on nested workflow ints/bools (JSON-3); paths confirmed campaign-relative (N-16) | 09.05, 09.06 |
| __manifest | int version n | n+1 | every shipped command annotated; doctor read vs --fix split; int `version` field documented as the manifest's own convention | 09.07 |

Quest JSON path fields move to campaign-relative plus `campaign_root` under the same N-16 task (09.06); quest surfaces are dev-only so the change is documented in the changelog without a formal contract version (none exists today).

### Notification path

Both, not either: (1) `docs/json-contracts.md` gains/updates a row per surface above including `workitem priority` (JSON-1) and a scope statement separating contract surfaces from best-effort ones (JSON-4); (2) one festival-app coordination workitem (`camp workitem create --type design` for the festival-app project) created in task 09.05 listing every bump, the version strings, and the new `profile` preflight field, so the app's bundled-binary update is planned work, not a surprise.

## Consequences

- festival-app keys on `schema_version` strings; old bundled binaries keep parsing old payloads until the app updates (camp is not publicly released; no external-consumer compat shims needed beyond the version strings themselves).
- The intent `--format json` deprecated alias prevents breaking the app's current invocation mid-transition.
- JSON-6's stability matrix in docs becomes the closeout artifact verified in 004_POLISH.
