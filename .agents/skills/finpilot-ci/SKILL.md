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
| `build-image.yml`             | push main + stable, manual        | Publish `:stable-testing` (main) or `:stable` (stable)        |
| `promote-main-to-stable.yml`  | push main, manual                 | Squash promotion PR `main` → `stable` (local impl)            |
| `sync-stable-to-main.yml`     | push stable                       | Merge direct `stable` hotfixes back to `main` (usually no-op) |
| `pr-validation.yml`           | PR → main                         | shellcheck + hadolint + pre-commit via `validate-pr`          |
| `renovate.yml`                | schedule 6h, push renovate config | Self-hosted Renovate runner                                   |
| `clean.yml`                   | schedule weekly                   | Delete GHCR images older than 90 days                         |
| `validate-brewfiles.yml`      | PR paths: `custom/brew/**`        | Homebrew Brewfile syntax check                                |
| `validate-flatpaks.yml`       | PR paths: `custom/flatpaks/**`    | Flathub app ID existence check                                |
| `validate-justfiles.yml`      | PR paths: `Justfile`              | `just --list` syntax check                                    |
| `validate-renovate.yml`       | PR paths: `.github/renovate.json` | `renovate-config-validator`                                   |

## Branch Promotion and Tags

- `main` is the testing branch and publishes `:stable-testing` (plus bare
  `:testing`, which the promotion release gate resolves).
- `stable` is the production branch and publishes `:stable`.
- Promotion uses a **local** squash workflow (`promote-main-to-stable.yml`)
  plus `reusable-sync-branches.yml` from `projectbluefin/actions`. pull[bot] /
  `.github/pull.yml` was rejected (issues #235/#237); do not add it.
- **Personal-account forks must not use `reusable-promote-squash.yml`.** It
  hardcodes `--reviewer <owner>/maintainers`; teams are org-only, so
  `requestReviewsByLogin` fails right after the PR is created and every
  promotion run exits 1 (plus a spurious "promotion conflict" issue). The
  local workflow drops the reviewer and calls the factory release gate
  (`reusable-release-gate.yml`) directly. Fix upstream before switching back.
- **`gh` in `run:` steps needs the token.** Add `GH_TOKEN: ${{ github.token }}`
  at the job level or every `gh` call fails with "To use GitHub CLI in a GitHub
  Actions workflow, set the GH_TOKEN environment variable."
- **`stable` is protected by a ruleset; promotion merges via PAT auto-merge.**
  The promotion workflow enables `gh pr merge --squash --auto` with
  `GH_TOKEN: ${{ secrets.PROMOTE_TOKEN }}` (a PAT). A PAT-performed merge is a
  normal push and fires the `:stable` build. Do NOT enable auto-merge with the
  runner token: GitHub suppresses workflow runs triggered by `GITHUB_TOKEN`
  events, so a GITHUB_TOKEN merge to `stable` produces NO `Build and Push
  Image` run and `:stable` silently goes stale (verified 2026-08-11: merged
  commit had zero workflow runs).
- **No merge queue on personal-account repos.** GitHub's merge queue is
  org-owned-repo only; the API rejects a `merge_queue` ruleset rule on a
  user-owned repo with `422 Invalid rule 'merge_queue'` (verified 2026-08-11),
  so the upstream `enqueuePullRequest` path (projectbluefin/bluefin
  `use_merge_queue: true`) cannot run here. Do not add a `merge_queue` rule to
  the `stable` ruleset; PAT auto-merge is the fork-side delivery mechanism.
- **Auto-merge races with mergeability.** Right after `gh pr
  create` the PR's mergeability is `UNKNOWN`; retry the enable in a
  loop (~60s). `gh pr merge --auto` also **requires a merge method flag**
  (`--squash`) when non-interactive. Auto-merge stays blocked until `validate`
  (the posted status) propagates.
- **Bot-created PRs park PR-triggered runs in `action_required`.** The promotion
  PR is created with the runner token, so `PR Validation` and
  `enforce workflow labels` runs on it are held by the platform approval gate
  (0 jobs, `conclusion: action_required`) until a maintainer approves. Harmless
  here: the promotion posts the synthetic `validate=success` status, which
  satisfies the ruleset's required check. Do NOT switch these to
  `pull_request_target` to "fix" it — that changes the security model.
- The conflict-issue auto-close must match **both** historical titles:
  `ci: main→stable promotion conflict` and `ci: testing→main promotion conflict`
  (the factory-reusable-era title).
- The `Determine image tag` step sets `TAG_STREAM=testing` off the production
  branch; `Finalize branch tags` renames `testing*` tags to `stable-testing-*`
  so they never collide with production `stable-daily*` aliases.
- The release gate verifies cosign signatures on `:testing`; keyless signing
  (optional step in `build-image.yml`) must be enabled for `release/ready`.

## Build Timeouts and the GHCR Layer Cache

The Justfile seeds builds with a registry layer cache:
`podman build --cache-from/--cache-to ghcr.io/<owner>/<image>` (gated on an
anonymous `skopeo list-tags` succeeding). Cache tags are content-hash-named
(`<repo>:<64-hex>`); hashes are **stable across runs** for identical inputs.

**Root cause (verified 2026-08-26):** Rootful podman on CI runners falls
back to **VFS** storage driver (native-overlay disabled, btrfs loopback
absent). VFS copies the entire ~12 GB rootfs per RUN step — buildah
commit takes 18-24 min per step, guaranteeing timeout at any reasonable
`timeout_minutes`. Rootless podman uses **fuse-overlayfs** with instant
commits (~3 s). The fix is to run all podman commands rootless (no sudo).

**CI workflow architecture (rootless):**
- Build: `just build` → rootless podman build with `--cache-from/--cache-to`
- Seed-pull: pre-pull last published image to populate blob-info cache
- Tag: inline `podman tag` (rootless)
- Push: inline `podman push` with retry + zstd:chunked + `skopeo copy` aliases
- Sign: inline `cosign sign` keyless (OIDC/Fulcio) + verify + legacy .sig check
- No `sudo` anywhere — no rootful storage involved
- No bridge step — everything shares rootless storage

Failure signature (Aug 18–25, 2026): every "Build and Push Image" run
fails at exactly `timeout_minutes`; then `nick-fields/retry` crashes with
`Error: kill EPERM` — the build runs under `sudo`, so the process tree is
root-owned and node cannot kill it. **`max_attempts` never retries.**

Key rules:

- **All podman commands must run rootless** on CI. Never add `sudo` to
  podman/buildah commands — VFS fallback is invisible and catastrophic.
- **Serialize main/stable builds** (`concurrency.group: github.workflow`,
  `cancel-in-progress: false`) so stable promotion inherits main's fresh
  cache layers instead of racing cold; queued runs must never cancel an
  in-progress production publish mid-push.
- **Recovering a missed `:stable`:** re-run the failed run (`gh run rerun`)
  once the cache works. Promotion PRs are already PAT-merged at that point;
  nothing retries automatically. Re-runs use the original commit's workflow
  definition — a fix merged afterwards does NOT apply to re-runs.
- **Renaming the image/repo resets the registry cache to empty** (new GHCR
  repo), and package visibility flips (public/private) change whether the
  anonymous `list-tags` gate passes at all — check both when builds suddenly
  go cold.
- `timeout_minutes: 240` fits one attempt inside GitHub's 360-min job
  ceiling; don't rely on retries (EPERM crash above).
- **Caller permissions are the ceiling.** Any workflow that calls a reusable
  workflow must grant a `permissions:` block that is a superset of every job
  permission the callee declares — an explicit caller block zeroes unlisted
  scopes. The promotion caller needs `packages: read` because the reusable
  release gate reads GHCR; without it GitHub rejects the run with
  `startup_failure` before any job starts (no log to diagnose). actionlint does
  not model this check — keep callee scopes in sync manually.

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
