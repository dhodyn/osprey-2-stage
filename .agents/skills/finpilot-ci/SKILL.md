---
name: finpilot-ci
description: >-
  GitHub Actions workflows, projectbluefin/actions composite actions,
  Renovate configuration, and PR validation for finpilot.
  Use when changing .github/workflows/, .github/renovate.json, or .hadolint.yaml.
---

# finpilot CI

## When to Use

- Editing any `.github/workflows/*.yml`
- Editing `renovate.json`
- Adding new tooling to `build-image.yml`
- Debugging CI failures
- Deciding what to automerge vs require review

## When NOT to Use

- Containerfile / Justfile / build script changes — use `finpilot-build`
- Runtime customisations — use `finpilot-custom`

## Core Process

1. **Identify the workflow responsible** for your change (see table below)
2. **Check `projectbluefin/actions`** to confirm the composite action exists and what inputs it takes
3. **Pin any new tool** with a specific version + Renovate tracking comment
4. **Validate** locally: `actionlint .github/workflows/*.yml`
5. **Do not widen automerge scope** beyond `digest/pin/pinDigest` for the broad rule

## Workflow Map

| File                          | Trigger                           | Purpose                                                       |
| ----------------------------- | --------------------------------- | ------------------------------------------------------------- |
| `build-image.yml`             | push main + PR → main, manual     | Build on PR (writes layer+DNF cache); publish `:stable` on merge |
| `pr-validation.yml`           | PR → main                         | shellcheck + hadolint + pre-commit via `validate-pr`          |
| `renovate.yml`                | schedule 6h, push renovate config | Self-hosted Renovate runner                                   |
| `clean.yml`                   | schedule weekly                   | Delete GHCR images older than 90 days                         |
| `validate-brewfiles.yml`      | PR paths: `custom/brew/**`        | Homebrew Brewfile syntax check                                |
| `validate-flatpaks.yml`       | PR paths: `custom/flatpaks/**`    | Flathub app ID existence check                                |
| `validate-justfiles.yml`      | PR paths: `Justfile`              | `just --list` syntax check                                    |
| `validate-renovate.yml`       | PR paths: `.github/renovate.json` | `renovate-config-validator`                                   |

## Single-Branch Model and Cache

- **`main` IS production.** There is no `stable` branch, no promotion PR, no
  PAT, no second ruleset, no merge queue. Merging a PR to `main` publishes
  `:stable` (plus `stable-daily-*` and version tags).
- **PRs are the gate.** A PR runs a full image build (`build` check) and, since
  forks are blocked by the guard step, can safely write both the registry layer
  cache (`REGISTRY_CACHE_WRITE: "1"`) and the DNF cache. The post-merge `main`
  build reuses that cache → fast download/assemble/sign/push, no recompilation.
- **Branch protection required checks use JOB names as contexts.** `main`
  requires `["validate", "Build and push image"]` — the build job's name, not
  `build`. A context that doesn't match any check run (e.g. `build`) leaves the
  PR `mergeStateStatus: BLOCKED` despite every check passing. Verify with
  `gh pr view N --json mergeStateStatus` before opening a support ticket.
- **Fork guard:** `Block fork PRs` runs first in `build-image.yml` and exits 1
  on any PR whose `head.repo.full_name != github.repository`. Public personal
  repos cannot disable forking, so this is the only defense against fork PRs
  writing to the shared cache with an untrusted tree.
- **Cache reuse is verified** (benchmarked 2026-08-12 on a pristine clone:
  cold build 269s vs warm build 36.5s with one package added, all unchanged
  layers `Using cache`). `preflight` runs `podman login` to ghcr.io with the
  `GITHUB_TOKEN` before the build on both push and PR runs, so `--cache-to`
  writes during a same-repo PR build work — no separate login step needed.
- The `Determine image tag` step is gone; `TAG_STREAM` is hardcoded to
  `"stable"` and `Finalize branch tags` just dedupes generated tags and
  guarantees the mutable `:stable` tag is always published.
- Keyless signing (the optional step in `build-image.yml`) is enabled; merges
  publish signed `:stable`. Verify with `cosign verify`.
- `gh` in `run:` steps needs the token. Add `GH_TOKEN: ${{ github.token }}`
  at the job level or every `gh` call fails with "To use GitHub CLI in a GitHub
  Actions workflow, set the GH_TOKEN environment variable."
- **Caller permissions are the ceiling.** Any workflow that calls a reusable
  workflow must grant a `permissions:` block that is a superset of every job
  permission the callee declares — an explicit caller block zeroes unlisted
  scopes. Without it GitHub rejects the run with `startup_failure` before any
  job starts (no log to diagnose). actionlint does not model this check — keep
  callee scopes in sync manually.

## Composite Action Pins

All actions from `projectbluefin/actions` are pinned to a **commit SHA**:

```yaml
uses: projectbluefin/actions/bootc-build/setup-runner@<sha> # v1
```

**Never use a floating tag like `@v1` or `@main`.** Renovate updates the SHA automatically.

The SHA comment (`# v1`) is for human readability only — Renovate ignores it.

## Rechunking

Set `ENABLE_RECHUNKING: "true"` in `build-image.yml` to enable the existing
`bootc-build/chunka` step. Keep the action active behind the feature flag rather
than commenting it out so Renovate continues to update its SHA.

The action is OCI-native and does not use `/usr/libexec/bootc-base-imagectl`.
Finpilot's default Fedora Silverblue image follows the RPM path, where chunkah
discovers components from the RPM database.

Rechunking is not a drop-in switch after replacing the default base with a
BuildStream-produced image. Those images require a generated `xattr-manifest`
because BuildStream strips component xattrs during OCI export.

Package cadence optimization is optional. `bootc-build/apply-pkg-intervals`
requires a maintained `files/pkg-intervals.tsv`; the reusable cadence workflow
also requires GitHub App credentials. Do not present either as a prerequisite
for basic rechunking.

## Adding a New Tool (e.g., jq, cosign)

Always pin to a specific version with a Renovate tracking comment:

```yaml
- name: Install <tool>
  env:
    # renovate: datasource=github-releases depName=owner/repo
    TOOL_VERSION: "1.2.3"
  run: |
    sudo wget -qO /usr/local/bin/<tool> \
      "https://github.com/owner/repo/releases/download/v${TOOL_VERSION}/<tool>-linux-amd64"
    sudo chmod +x /usr/local/bin/<tool>
```

The `renovate.json` custom manager tracks this pattern:

```json
{
  "customType": "regex",
  "description": "Track pinned tool versions in workflow env vars",
  "managerFilePatterns": ["/^\\.github\\/workflows\\/.+\\.yml$/"],
  "matchStrings": [
    "# renovate: datasource=(?<datasource>[^\\s]+) depName=(?<depName>[^\\s]+)\\n\\s+\\w+: \"(?<currentValue>[^\"]+)\""
  ]
}
```

Never use `/releases/latest/` — it is non-reproducible.

## Renovate Automerge Scope

### ✅ Safe to automerge broadly (digest/pin only)

```json
{
  "matchUpdateTypes": ["digest", "pin", "pinDigest"],
  "automerge": true
}
```

Digest-only updates are hash changes with no API surface change. Safe.

### ✅ Safe to automerge for trusted first-party actions

```json
{
  "matchPackageNames": ["projectbluefin/actions"],
  "matchUpdateTypes": ["digest", "pinDigest", "pin", "patch", "minor"],
  "automerge": true
}
```

`projectbluefin/actions` is controlled by the same factory — minor/patch bumps are safe.

### ❌ Do NOT automerge broadly for `minor`/`patch`

Minor and patch updates across all packages can change workflow behaviour or introduce
regressions. They require human review before merging to an OS image template that ships
to users' machines.

## Renovate OCI Digest Tracking

All OCI image digests are pinned inline in `Containerfile` `FROM` lines and
tracked by Renovate's built-in `dockerfile` manager — pinning pattern:
`finpilot-build`.

When Renovate updates a digest it opens a PR that changes only the relevant
`Containerfile` line. The next CI build uses it directly.

## Renovate Workflow Requirements

The self-hosted Renovate runner requires a `RENOVATE_TOKEN` (Classic PAT with
`repo` + `workflow` scopes; creation: `finpilot-onboarding`). The
`check-token-health` composite action validates it at the start of the workflow,
so a missing or expired token fails the workflow **before** running Renovate —
not midway through.

## hadolint Config (.hadolint.yaml)

Suppressions are documented with reasons:

```yaml
ignore:
  - DL3006 # Commented-out alternative FROM lines use ARG interpolation
  - DL3059 # Multiple consecutive RUN — intentional design (cache layering)
  - SC2312 # Style preference — command substitution in conditions
```

Add suppressions sparingly. If you suppress a new rule, document the reason inline.

## Common Rationalizations

| Rationalization                                          | Reality                                                                                |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| "I'll use `/releases/latest/` for now and pin it later." | You won't. Non-reproducible builds silently fail months later. Pin immediately.        |
| "Minor/patch automerge is fine — it's just a template."  | Templates ship to users' machines. A bad automerge in a CI action can break all forks. |
| "I don't need Renovate tracking for this one tool."      | Unpinned tools silently break when upstream releases a breaking change.                |

## Red Flags

- Tool installed via `/releases/latest/` without version pin
- Automerge rule includes `minor` or `patch` for all packages (`matchPackageNames` not scoped)
- Composite action used with a floating tag (`@v1`, `@main`) instead of a commit SHA
- `GITHUB_TOKEN` used as the Renovate token (it cannot open PRs to other repos)
- `renovate.json` changed without running `renovate-config-validator`

## Verification

- [ ] Every `uses:` in workflows is pinned to a commit SHA with a version comment?
- [ ] Every new tool install has a pinned version + `# renovate: datasource=...` comment?
- [ ] Automerge broad rule is `digest/pin/pinDigest` only (not `minor`/`patch`)?
- [ ] `actionlint .github/workflows/*.yml` passes clean?
- [ ] `renovate-config-validator .github/renovate.json` passes clean?
- [ ] `RENOVATE_TOKEN` secret documented in SETUP_CHECKLIST.md?
