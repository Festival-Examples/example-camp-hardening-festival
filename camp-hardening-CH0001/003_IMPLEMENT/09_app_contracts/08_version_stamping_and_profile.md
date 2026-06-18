---
fest_type: task
fest_id: 08_version_stamping_and_profile.md
fest_name: version_stamping_and_profile
fest_parent: 09_app_contracts
fest_order: 8
fest_status: completed
fest_autonomy: medium
fest_created: 2026-06-11T23:27:25.752008-06:00
fest_updated: 2026-06-15T01:43:19.036668-06:00
fest_tracking: true
---


# Task: Version stamping, Profile field, and `camp version --json` hardening (R21 / BR-3 / BR-4 / N-25 / UX-15 / JSON-5)

## Objective

Thread ldflags version stamping into the cross-platform build recipes, add a `debug.ReadBuildInfo()` fallback for `go install` binaries, introduce a `Profile` string constant via a build-tag pair so festival-app can preflight dev-only commands, surface `profile` and `schema_version` in `camp version --json`, fix `--short`/`--json` flag precedence, and convert `versionCmd` from `Run` to `RunE`.

## Requirements

- [ ] Cross-platform build recipes (`linux`, `darwin`, `darwin-arm64`) in `.justfiles/build.just` pass version ldflags (deriving `VERSION` from the git tag or the root justfile `version` variable); the `windows` recipe is NOT updated (unsupported platform, per project policy)
- [ ] `internal/version/version.go` has a `debug.ReadBuildInfo()` fallback so `go install .../cmd/camp@vX.Y.Z` reports the module version rather than `dev`
- [ ] A `Profile` string variable exists in `internal/version/` set by a build-tag constant pair: `profile_dev.go` (`//go:build dev`, sets `Profile = "dev"`) and `profile_stable.go` (`//go:build !dev`, sets `Profile = "stable"`)
- [ ] `camp version --json` emits `schema_version`, `profile`, and snake_case fields per D004; the existing camelCase keys may be kept as aliases or dropped (implementer picks and documents the decision; snake_case is canonical since camp is not publicly released)
- [ ] `--short` does NOT silently win over `--json`; when both flags are provided, either `--json` wins (preferred, more machine-useful) or the command returns an error; document the chosen rule in the Long help
- [ ] `versionCmd` uses `RunE` not `Run`; a JSON marshal error exits non-zero

## Implementation

### Background

`justfile` line 14 (verified at worktree bc2ad1f):
```
ldflags := "-X " + version_pkg + ".Version=" + version + " -X " + version_pkg + ".Commit=" + commit + " -X " + version_pkg + ".BuildDate=" + build_date
```
This variable is defined but referenced by ZERO recipes. The cross-platform recipes in `.justfiles/build.just` lines 69-104 use `go build -ldflags '-s -w'` only (or `-tags '{{build_tags}}' -ldflags '-s -w'`). So all cross-platform binaries report `camp dev (built unknown, commit unknown)`.

`internal/version/version.go` (verified at worktree bc2ad1f):
- Lines 7-16: `var Version = "dev"`, `Commit = "unknown"`, `BuildDate = "unknown"`
- Lines 19-36: `Info` struct with camelCase JSON fields (`buildDate`, `goVersion`); no `schema_version`, no `profile`
- No `debug.ReadBuildInfo()` call anywhere

`cmd/camp/version.go` (verified at worktree bc2ad1f):
- Line 20: `Run:` (not `RunE:`)
- Lines 26-38: `short` is checked first; if `--short --json` both set, `short` wins silently

### Step 1: fix cross-platform ldflags

In `.justfiles/build.just`, update the `linux`, `darwin`, and `darwin-arm64` recipes to pass the ldflags variable. The root justfile already defines `ldflags` correctly. In the module system, the recipes need access to the root variable. The `build_tags` variable is already threaded through via `{{build_tags}}`; apply the same pattern for `{{ldflags}}` or replicate the logic:

```just
[no-cd]
linux:
    #!/usr/bin/env bash
    set -euo pipefail
    version=$(git describe --tags --abbrev=0 2>/dev/null || echo "dev")
    commit=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
    build_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    ldflags="-s -w -X github.com/Obedience-Corp/camp/internal/version.Version=${version} \
             -X github.com/Obedience-Corp/camp/internal/version.Commit=${commit} \
             -X github.com/Obedience-Corp/camp/internal/version.BuildDate=${build_date}"
    if [ -n "{{build_tags}}" ]; then
        GOOS=linux GOARCH=amd64 go build -tags '{{build_tags}}' -ldflags "${ldflags}" \
          -o {{bin_dir}}/{{binary_name}}-linux-amd64 ./cmd/camp
    else
        GOOS=linux GOARCH=amd64 go build -ldflags "${ldflags}" \
          -o {{bin_dir}}/{{binary_name}}-linux-amd64 ./cmd/camp
    fi
```

Apply the same pattern to `darwin` and `darwin-arm64`. Do not modify `windows` (unsupported per camp/fest no-Windows policy).

Alternatively, if the root `ldflags` variable is accessible inside module recipes via `{{ldflags}}`, use that directly. Test one recipe manually to confirm the stamping works before applying to all three.

### Step 2: add `debug.ReadBuildInfo()` fallback

In `internal/version/version.go`, add the fallback in `init()`:

```go
import (
    "runtime"
    "runtime/debug"
)

func init() {
    if Version == "dev" {
        if info, ok := debug.ReadBuildInfo(); ok && info.Main.Version != "" {
            Version = info.Main.Version
        }
    }
    if Commit == "unknown" {
        if info, ok := debug.ReadBuildInfo(); ok {
            for _, s := range info.Settings {
                if s.Key == "vcs.revision" && len(s.Value) >= 7 {
                    Commit = s.Value[:7]
                    break
                }
            }
        }
    }
}
```

Or call `debug.ReadBuildInfo()` once and cache the result. This ensures that `go install .../cmd/camp@v1.2.3` reports `v1.2.3` instead of `dev`.

### Step 3: add Profile via build-tag pair

Create two new files:

`internal/version/profile_dev.go`:
```go
//go:build dev

package version

// Profile is "dev" when built with the dev build tag.
const Profile = "dev"
```

`internal/version/profile_stable.go`:
```go
//go:build !dev

package version

// Profile is "stable" when built without the dev build tag.
const Profile = "stable"
```

The existing `release_profile_dev_test.go` and `release_profile_stable_test.go` test the profile-gating pattern; check those files for the test structure to follow.

### Step 4: update `Info` struct and `camp version --json`

In `internal/version/version.go`, update `Info` and `Get()`:

```go
type Info struct {
    SchemaVersion string `json:"schema_version"`
    Version       string `json:"version"`
    Commit        string `json:"commit"`
    BuildDate     string `json:"build_date"`
    GoVersion     string `json:"go_version"`
    Platform      string `json:"platform"`
    Profile       string `json:"profile"`
}

func Get() Info {
    return Info{
        SchemaVersion: "version/v1alpha1",
        Version:       Version,
        Commit:        Commit,
        BuildDate:     BuildDate,
        GoVersion:     runtime.Version(),
        Platform:      runtime.GOOS + "/" + runtime.GOARCH,
        Profile:       Profile,
    }
}
```

Snake_case is canonical (D004 decision: camp is not publicly released, so a clean break from camelCase is simpler than dual-key aliases). Document this in a comment in the struct. If keeping old camelCase keys for backward compat is preferred, add them with `json:"buildDate"` etc. alongside the snake_case versions; the implementer picks and notes the choice in `docs/json-contracts.md`.

### Step 5: fix `--short`/`--json` precedence and convert to `RunE`

In `cmd/camp/version.go`, apply:

```go
var versionCmd = &cobra.Command{
    Use:   "version",
    Short: "Show version information",
    Long: `Show camp version, build information, and runtime details.

When both --short and --json are provided, --json wins.

Examples:
  camp version           Show full version info
  camp version --short   Show only version number
  camp version --json    Output as JSON`,
    RunE: func(cmd *cobra.Command, args []string) error {
        short, _ := cmd.Flags().GetBool("short")
        jsonOut, _ := cmd.Flags().GetBool("json")

        info := version.Get()

        if jsonOut {
            out, err := json.MarshalIndent(info, "", "  ")
            if err != nil {
                return camperrors.Wrap(err, "marshal version info")
            }
            _, err = fmt.Fprintln(cmd.OutOrStdout(), string(out))
            return err
        }

        if short {
            _, err := fmt.Fprintln(cmd.OutOrStdout(), info.Version)
            return err
        }

        fmt.Fprintf(cmd.OutOrStdout(), "camp %s\n", info.Version)
        fmt.Fprintf(cmd.OutOrStdout(), "commit: %s\n", info.Commit)
        fmt.Fprintf(cmd.OutOrStdout(), "built: %s\n", info.BuildDate)
        fmt.Fprintf(cmd.OutOrStdout(), "go: %s\n", info.GoVersion)
        fmt.Fprintf(cmd.OutOrStdout(), "platform: %s\n", info.Platform)
        return nil
    },
}
```

Note `--json` checked before `--short` so `--short --json` gives JSON output.

### Tests

1. Unit tests for `version.Get()`:
   - Stable profile build: `info.Profile == "stable"`.
   - Dev profile build (`-tags dev`): `info.Profile == "dev"`.
   - `info.SchemaVersion == "version/v1alpha1"`.

2. Command test for `versionCmd`:
   - `--json` output is valid JSON with `profile` and `schema_version` fields.
   - `--short --json` returns JSON (not just the version string).
   - Marshal error returns non-zero (hard to test directly; mock or accept as documented).

The existing `cmd/camp/release_profile_dev_test.go` and `release_profile_stable_test.go` test build-tag gating; add profile-assertion rows there.

## Done When

- [ ] All requirements met
- [ ] `linux`, `darwin`, `darwin-arm64` cross-platform recipes stamp version ldflags; `windows` unchanged
- [ ] `go install` binary reports module version via `debug.ReadBuildInfo()` fallback
- [ ] `profile_dev.go` and `profile_stable.go` files exist with correct build tags
- [ ] `camp version --json` emits `schema_version`, `profile`, snake_case fields
- [ ] `--json` wins over `--short` when both provided
- [ ] `versionCmd` uses `RunE`; marshal error exits non-zero
- [ ] Both-profile gate passes (tests assert `profile` field value differs per profile)
- [ ] `docs/json-contracts.md` updated with version surface row and snake_case convention decision