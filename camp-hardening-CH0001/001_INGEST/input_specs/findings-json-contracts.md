# Findings: JSON Output Contracts (festival-app Integration Surface)

Context: festival-app shells out to bundled dev-profile camp binaries and parses JSON output, most heavily `camp workitem ... --json`. This file documents what camp actually emits and where schema discipline holds or breaks.

---

## What internal/jsoncontract actually enforces

`internal/jsoncontract` enforces ONLY the error envelope (`schema_version` plus `error{code,message,hint,exit_code}` on stderr) and `--json` detection (`Requested` with an argv fallback that correctly honors `--json=false`, well tested in `internal/jsoncontract/errors_test.go`). Success payload shapes are NOT enforced by it; each command defines inline structs. It is used consistently across the `workitem` and `workflow` command families only (19 files); intent, version, list, status, doctor, leverage, worktrees, and skills bypass it entirely. That matches the documented scope of `docs/json-contracts.md`, but it means "supports --json" does not imply "has the error contract" (cross-referenced as UX-21).

## Schemas festival-app depends on (verified in code)

- **`camp workitem --json`:** `workitems/v1alpha5` (`internal/workitem/json.go:16`): `{schema_version, generated_at, campaign_root, sort{primary,secondary,direction}, items[], counts{total,intent,design,explore,festival}}`. Items (`internal/workitem/model.go:34-52`): `key, workflow_type` (lowercase enum intent|design|explore|festival), `lifecycle_stage, title, relative_path, primary_doc, item_kind` (file|directory), `created_at/updated_at/sort_timestamp` (RFC3339Nano), `manual_priority` (omitempty), `summary, source_id, source_metadata` (map), `stable_id` (omitempty), `workflow` (omitempty pointer). Versioned via documented changelog, never-null items array. **This is the model contract; discipline here is good.**
- **`camp workitem priority --json`:** `workitem-priority/v1alpha1` (`internal/commands/workitem/json_contract.go:9`): `{schema_version, generated_at, key, priority, cleared}`. Clear emits `"priority": ""`. Implementation quality of the new command is high (camperrors, jsoncontract.RunE plus Args plus FlagErrorFunc, selector reuse, validation in `parsePriorityLevel` at `priority.go:55-70`).
- Other workitem subcommands each have a named version constant (`internal/commands/workitem/json_contract.go:5-15`, `internal/workitem/links/types.go:17-18`); the workflow family uses `workflow/v1`.

---

## JSON-1. docs/json-contracts.md is missing the priority surface (MEDIUM)

The contract table (`docs/json-contracts.md:15-32`) lists every other workitem surface but not `camp workitem priority --json`, despite it being built explicitly for festival-app automation (PR #318). Add the row, including the cleared semantics.

## JSON-2. Intent JSON is an unversioned ad hoc contract (MEDIUM)

- `cmd/camp/intent/list.go:213-231` emits a bare top-level array of inline structs with no `schema_version`; `cmd/camp/intent/find.go:166-173` and `cmd/camp/intent/count.go:102-107` likewise.
- It is also triggered via `-f json` / `--format json` rather than `--json`, so the jsoncontract error envelope never applies; failures emit plain text.
- **Why it matters:** festival-app's intent capture UX (IC0001 work) and any agent parsing intents has no version pin; the shape can drift silently. The prior festival-app audit already found intent contract drift bugs on the app side.
- **Recommendation:** wrap intent list/find/count/show in a versioned envelope like the workitem payload and migrate to `--json` with the error contract.

## JSON-3. omitempty hazards inside WorkItemWorkflow (MEDIUM-LOW)

- `internal/workitem/model.go:57-64`: `current_step`, `total_steps`, `completed_steps` are `int,omitempty` and `blocked`, `doc_hash_changed` are `bool,omitempty`. A consumer cannot distinguish "step 0 / not blocked" from "field absent"; a workflow legitimately at 0 completed steps loses the field. The changelog calls this intentional for byte-compat.
- **Recommendation:** at the next schema_version bump, drop omitempty inside the nested struct; the `workflow` pointer already gates presence.

## JSON-4. Unversioned or convention-violating side surfaces (LOW)

- `camp version --json` uses camelCase (`buildDate`, `goVersion`; `internal/version/version.go:20-26`) and no schema_version, violating the snake_case rule at `docs/json-contracts.md:38`.
- `camp status all --json` (`cmd/camp/status_all.go:60-79`) is snake_case but unversioned; it also emits a human string instead of JSON in the no-submodules case (UX-20), which is a parser-breaking bug for any consumer.
- `__manifest` (`cmd/camp/manifest.go:12-22`) uses an int `version` field rather than the string `schema_version` convention; festival-app keys on the manifest for agent-allowed gating, so the divergence should at least be documented.
- `cmd/camp/list.go:24`: `LastAccess time.Time` with `json:"last_access,omitempty"`; omitempty is a no-op on time.Time so zero values serialize as `0001-01-01T00:00:00Z`. Use `omitzero` or a pointer.
- **Recommendation:** scope `docs/json-contracts.md` explicitly (which surfaces are contract, which are best-effort), then version the ones festival-app reads.

## JSON-5. No channel field anywhere in machine output (MEDIUM, cross-ref BR-4)

festival-app bundles dev-profile camp on purpose, and dev-only commands (`quest`, `flow`) are part of what the app calls. No JSON surface (version, manifest) reports the build profile, so the app cannot preflight "does this binary have quest". Add `profile` to `camp version --json` and/or the manifest.

## JSON-6. Stability assessment for festival-app (summary)

| Surface | Versioned | Error envelope | Risk |
|---|---|---|---|
| workitem list/show/current/links | yes (v1alpha5 family) | yes | low; omitempty nuance (JSON-3) |
| workitem priority | yes (v1alpha1) | yes | low; missing from contract doc (JSON-1) |
| workflow family | yes (workflow/v1) | yes | low |
| intent list/find/count/show | no | no | HIGH drift risk (JSON-2) |
| status all | no | no | medium; empty-case corruption (UX-20) |
| doctor | no | no | medium; exit-0-on-failure (UX-11) makes it actively misleading |
| version | no | no | low |
| __manifest | int version | n/a | low; convention divergence |
