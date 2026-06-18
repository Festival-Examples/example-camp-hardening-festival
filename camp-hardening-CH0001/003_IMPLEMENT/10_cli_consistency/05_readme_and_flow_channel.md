---
fest_type: task
fest_id: 05_readme_and_flow_channel.md
fest_name: readme_and_flow_channel
fest_parent: 10_cli_consistency
fest_order: 5
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.76993-06:00
fest_updated: 2026-06-15T02:45:51.302742-06:00
fest_tracking: true
---


# Task: README Install Path and Flow Channel Fixes

## Objective

Fix the broken `go install` command in `README.md`, make scaffolded skills and templates channel-aware so stable-initialized campaigns never instruct `camp flow` commands, add a dev-channel caveat for `flow` in `README.md`, drop the `camp flow init` hint from `dungeon add`'s Long help, and pin `BUILD_TAGS=''` in the `docs` justfile recipe to prevent dev-only pages leaking into the CLI reference.

## Requirements

- [ ] **DEPENDS ON sequence 09 task 08**: the `Profile` build-tag constant pair must exist before this task runs. Verify by checking that `internal/version/` (or wherever task 09.08 places it) has a `const Profile = "dev"` file and a `const Profile = "stable"` file guarded by `//go:build dev` and `//go:build !dev` respectively.
- [ ] `README.md:28`: the `go install github.com/Obedience-Corp/camp@latest` command is corrected to `go install github.com/Obedience-Corp/camp/cmd/camp@latest`.
- [ ] `README.md`: any section that mentions `camp flow` as a generally available command adds a parenthetical noting it is the dev channel (e.g., `(dev channel only; not available in stable builds)`). The review cited lines 15 and 206; verify and update those.
- [ ] `internal/scaffold/campaign/templates/.campaign/skills/campaign-workflows/SKILL.md.tmpl:33-35`: the `camp flow status`, `camp flow move`, and `camp flow sync` lines are replaced with a Go template conditional that includes those lines only when the scaffolding binary is dev-profile, using the `Profile` constant:

```
{{- if eq .Profile "dev"}}
## Flow (dev channel)

```bash
camp flow status
camp flow move <item> <status>
camp flow sync
```
{{- end}}
```

The template data struct passed to this template must include a `.Profile` field. Read the scaffolding code in `internal/scaffold/` to understand how templates are rendered and where the data struct is constructed; add `.Profile` to that struct and populate it with `version.Profile` (the constant from task 09.08).

- [ ] `internal/scaffold/campaign/templates/dungeon/OBEY.md.tmpl:36-42`: the `camp flow move` lines and the "Finishing Work" / "Deferring Work" sections that instruct `camp flow move` are conditioned the same way using the `.Profile` field in the template data, or the flow-specific sections are removed entirely and replaced with generic guidance that works for both profiles.
- [ ] `cmd/camp/dungeon/add.go`: the Long help text (verified at line 26: "This creates the same dungeon structure as 'camp flow init' but without...") is updated to remove or rephrase the `camp flow init` reference. The replacement should describe what `dungeon add` actually does without referencing a dev-only command. Example: change to "Initialize the dungeon directory structure directly, without requiring workflow setup."
- [ ] `justfile` `docs` recipe (verified near line 123: `docs: build-camp` then the gendocs invocation): the recipe is updated to pin `BUILD_TAGS=''` so it always generates docs from the stable profile binary regardless of the shell's ambient `BUILD_TAGS`. Use just's recipe syntax to override the variable:

```just
docs: build-camp
    BUILD_TAGS='' ./{{bin_dir}}/{{binary_name}} gendocs --output docs/cli-reference --format markdown --single
```

Or if the recipe uses the binary built by `build-camp` which already obeys `BUILD_TAGS`, change the recipe to rebuild with empty tags:

```just
docs:
    BUILD_TAGS='' just build-camp
    ./{{bin_dir}}/{{binary_name}} gendocs --output docs/cli-reference --format markdown --single
```

Read the current recipe before editing.

- [ ] Both-profile gate passes after this task.

## Implementation

### Background

This task implements decision D002 (channel-aware scaffolding, Option B). The flow commands `camp flow status`, `camp flow move`, and `camp flow sync` are dev-only (`internal/commands/release/register_dev.go`). Every campaign initialized with a stable binary gets `SKILL.md.tmpl` rendered into `.campaign/skills/campaign-workflows/SKILL.md`, whose instructions return "unknown command" when the agent tries them. This is a first-run blocker for the `<5 minute` launch goal.

The `Profile` constant from sequence 09 task 08 follows the same build-tag pattern as the release commands. There will be two files:
- `internal/version/profile_dev.go` (tagged `//go:build dev`): `const Profile = "dev"`
- `internal/version/profile_stable.go` (tagged `//go:build !dev`): `const Profile = "stable"`

The scaffolding templates are rendered with a data struct at init time. Read `internal/scaffold/campaign/scaffold.go` (or the equivalent file where templates are rendered) to find the data struct. It likely has fields like `CampaignName`, `CampaignID`, etc. Add `.Profile string` and populate it from `version.Profile`.

Note: verify all cited line numbers against the worktree at `bc2ad1f` before editing.

### Verified file:line targets (as of review citations; verify before editing)

- `README.md:28` - broken `go install` command
- `README.md:15,206` - `camp flow` advertising without channel caveat
- `internal/scaffold/campaign/templates/.campaign/skills/campaign-workflows/SKILL.md.tmpl:33-35` - flow command instructions
- `internal/scaffold/campaign/templates/dungeon/OBEY.md.tmpl:36-42` - flow move instructions
- `cmd/camp/dungeon/add.go:26` - "camp flow init" reference in Long help
- `justfile:123-124` - `docs` recipe (approximate; read to verify)

### Step-by-step approach

**Step 1: Confirm the `Profile` constant exists.**

Read `internal/version/` directory. Confirm there is a `profile_dev.go` and `profile_stable.go` (or equivalently named files) with `const Profile = "dev"` / `"stable"`. If these files do not exist, stop and ensure sequence 09 task 08 has landed before proceeding with this task. The rest of this task depends on them.

**Step 2: Fix `README.md:28`.**

Read `README.md`. Find the Go Install section near line 28. Change:

```bash
go install github.com/Obedience-Corp/camp@latest
```

to:

```bash
go install github.com/Obedience-Corp/camp/cmd/camp@latest
```

**Step 3: Add flow channel caveat to `README.md`.**

Read lines 10-20 and 200-210 (approximate per review citations 15 and 206). For each mention of `camp flow` as a feature:
- If the mention implies general availability, add `(dev channel; requires `BUILD_TAGS=dev` build or dev-profile binary)` or a similar parenthetical.
- If there is a dedicated "Flow" section, add a note at the top: "This command is available in dev-profile binaries only."

**Step 4: Add `.Profile` to the scaffold data struct.**

Find the template rendering code. Start from `internal/scaffold/campaign/` and look for a function that calls `tmpl.Execute(...)` or `template.ParseFS(...)`. The data struct passed to template execution will need a `Profile string` field. For example, if the struct is `scaffoldData`, add:

```go
type scaffoldData struct {
    // existing fields...
    Profile string
}
```

And populate it:

```go
data := scaffoldData{
    // existing field population...
    Profile: version.Profile,
}
```

Import `"github.com/Obedience-Corp/camp/internal/version"`. Run both-profile builds immediately after this change to confirm the templates compile correctly in both profiles.

**Step 5: Conditionalize the flow instructions in `SKILL.md.tmpl`.**

Read the file. The lines around 33-35 currently are:

```
camp flow status
camp flow move <item> <status>
camp flow sync
```

Wrap them in a template conditional. The exact section depends on context; read the surrounding structure. A conservative change that adds a block without removing the surrounding markdown:

```
{{- if eq .Profile "dev"}}
## Flow (dev channel)

Use `camp flow` to manage workflow state:

```bash
camp flow status
camp flow move <item> <status>
camp flow sync
```
{{- end}}
```

If the existing section header is "## Flow", keep it inside the conditional so stable users do not see an empty section.

**Step 6: Conditionalize or remove flow instructions in `OBEY.md.tmpl`.**

Read lines 36-42 (approximate). The "Finishing Work" and "Deferring Work" sections instruct `camp flow move`. Two options:
- Wrap the entire section in `{{if eq .Profile "dev"}}...{{end}}` (matches step 5 approach).
- Replace with channel-neutral guidance: "Move items to `dungeon/completed/` or `dungeon/archived/` directories manually or via `camp dungeon move`."

The second option is better for stable users because it tells them what to do. Use the second option unless the template data already supports the conditional cleanly.

**Step 7: Fix `cmd/camp/dungeon/add.go` Long help.**

Read `add.go`. The Long help near line 26 says:

```
This creates the same dungeon structure as 'camp flow init' but without
the full workflow (no .workflow.yaml, active/, or ready/ directories).
```

Change to:

```
Initialize the dungeon directory structure without workflow setup.
Use this when you only need a dungeon for idea capture or temporary holding,
without requiring a full workflow (no .workflow.yaml, active/, or ready/ directories).
```

Or similar wording that describes what the command does without referencing `camp flow init` (a dev-only command that stable users do not have).

**Step 8: Pin `BUILD_TAGS` in the `docs` recipe.**

Read the `justfile` near line 123. The current recipe likely runs `build-camp` which honors the ambient `BUILD_TAGS`. Update it to pin an empty tags value:

```just
docs:
    BUILD_TAGS='' just build-camp
    ./{{bin_dir}}/{{binary_name}} gendocs --output docs/cli-reference --format markdown --single
```

Or if the recipe uses `{{build_camp}}` or equivalent, prefix the gendocs invocation with `BUILD_TAGS=''`. Read the just recipe syntax carefully before editing; just has its own variable scoping rules.

**Step 9: Stable-profile scaffold test.**

Write a test in `internal/scaffold/` (host `t.TempDir()` convention) that:
1. Calls the scaffold function with `Profile = "stable"` and verifies the rendered `.campaign/skills/campaign-workflows/SKILL.md` does not contain the string `camp flow`.
2. Calls the scaffold function with `Profile = "dev"` and verifies the rendered skill file DOES contain `camp flow`.

Since the `Profile` constant is a build-tag-driven constant (not a runtime argument), the test needs to either accept a `profile` parameter in the scaffold data struct (which this task adds) or use a test-specific override. The cleanest approach: write two test files, one for each profile, each with the appropriate build tag and asserting the appropriate output.

### Edge cases

**Template rendering may not use Go's `text/template`:** If the scaffold uses a different rendering mechanism (e.g., string replacement), the `{{if eq .Profile "dev"}}` syntax will not work and a different approach is needed. Read the rendering code before writing the template conditionals.

**SKILL.md.tmpl may be embedded:** The templates are under `internal/scaffold/campaign/templates/` which is likely embedded via `go:embed`. The conditional template syntax is rendered at runtime, not at embed time, so build-tag conditionals in the template file itself are not the right approach. The `{{if eq .Profile "dev"}}` Go template conditional is the right approach.

**`docs/cli-reference/camp_dungeon_add.md`:** This is a generated file (via `just docs`). After fixing the Long help in `add.go` and the `docs` recipe, regenerate the reference: `BUILD_TAGS='' just docs`. Commit the updated generated file.

### Out of scope

- Moving `flow` to stable is explicitly out of scope per D002.
- `camp flow history` unimplemented advertisement (ARCH finding) is sequence 12.02.
- BR-5's stable-build quest scaffolding half-in/half-out is sequence 11.06.

## Done When

- [ ] All requirements met
- [ ] `go install github.com/Obedience-Corp/camp/cmd/camp@latest` is the README install command
- [ ] A newly scaffolded campaign with a stable-profile binary has no `camp flow` references in `.campaign/skills/campaign-workflows/SKILL.md`
- [ ] A newly scaffolded campaign with a dev-profile binary DOES have `camp flow` instructions in the skill file
- [ ] `camp dungeon add --help` no longer mentions `camp flow init`
- [ ] `just docs` with no `BUILD_TAGS` set produces docs from the stable binary (verify by checking that the output does not contain quest or flow subcommand pages)
- [ ] Both-profile gate passes: `just build`, `BUILD_TAGS=dev just build`, `just test unit`, `BUILD_TAGS=dev just test unit`, `just lint`, `BUILD_TAGS=dev just lint`