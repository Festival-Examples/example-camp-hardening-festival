# Sequence 08 Code Review

Reviewed range: `c15601c^..59021fd`

## Findings

Critical issues: none.

Suggestions: none required before sequence completion.

## Scope Reviewed

- `camp run <project> ...` now dispatches project `just` calls with direct argv.
- Flow runner extra args are passed as shell positionals instead of interpolated command text.
- Workitem list and TUI refresh paths no longer prune/write priority state during reads.
- Priority pruning remains on explicit write/fix paths.
- Intent read commands warn about legacy `workflow/intents/` state without migration.
- `camp status all` no longer writes an unused cache file.

## Gate Evidence

- `GOFLAGS=-buildvcs=false just build stable`
- `GOFLAGS=-buildvcs=false just build dev`
- `GOFLAGS=-buildvcs=false just test unit`
- `GOFLAGS=-buildvcs=false BUILD_TAGS=dev just test unit`
- `GOFLAGS=-buildvcs=false just lint`
- `GOFLAGS=-buildvcs=false BUILD_TAGS=dev just lint`
- `GOFLAGS=-buildvcs=false just vet`
- `GOFLAGS=-buildvcs=false BUILD_TAGS=dev just vet`
- `GOFLAGS=-buildvcs=false just test integration` passed 373/373 tests.
