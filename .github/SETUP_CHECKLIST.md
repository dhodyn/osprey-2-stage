# Repository Setup Checklist

## Initial Setup

### 1. Rename Template

- [ ] Update `finpilot` to your name in **7 files** (use the `finpilot-templates` skill):
  1. `Containerfile` — `ARG IMAGE_NAME` and `ARG IMAGE_VENDOR`
  2. `Justfile` — `export IMAGE_NAME`
  3. `README.md` — title
  4. `artifacthub-repo.yml` — `repositoryID`
  5. `custom/ujust/README.md` — bootc switch example
  6. `.github/workflows/clean.yml` — `packages`
  7. `iso/iso.toml` — bootc switch URL

**Agent skills:** `finpilot-templates` (rename rules), `finpilot-onboarding` (fork bootstrap)

### 2. Enable GitHub Actions

- [ ] Settings → Actions → General → Enable workflows
- [ ] Set "Read and write permissions"

### 3. Configure the Single-Branch Model

This template uses a **single-branch model**: `main` IS production. Every
change lands via a PR to `main`; merging publishes `:stable` (with the
`stable-daily-*` and version tags). There is no `stable` branch, no promotion
PR, and no `PROMOTE_TOKEN`.

- [ ] Enable **Settings → Actions → General → "Allow GitHub Actions to create
      and approve pull requests"** — required for Renovate auto-merge

- [ ] Configure branch protection for `main`:
  - Settings → Branches → Add rule
  - Set **Branch name pattern** to `main`
  - Enable **"Require a pull request before merging"**
  - Enable **"Require status checks to pass before merging"**
  - Add **`validate`** and **`build`** as required status checks
  - Enable "Require branches to be up to date before merging" (optional)
- [ ] Merge button (Settings → General → Pull requests): allow **merge commits,
      rebase, and squash** — merge method is context-driven per PR
- [ ] Keep **"Allow auto-merge"** and **"Allow GitHub Actions to create and
      approve pull requests"** enabled (Renovate needs both)
- [ ] Fork PRs are blocked by the guard step in `build-image.yml` — public
      personal repos cannot disable forking, so the guard fails the `build`
      check on fork PRs (they would write to the shared layer/DNF cache with
      an untrusted tree)

**Migrating an existing two-branch fork** (had `stable` + promotion): after
merging this single-branch change, tear down the old machinery last, one step
at a time:

```bash
gh api -X DELETE repos/OWNER/REPO/git/refs/heads/stable
gh api -X DELETE repos/OWNER/REPO/git/refs/heads/auto/promote-main-to-stable
gh api -X DELETE repos/OWNER/REPO/rulesets/20697458        # stable release protection
gh secret delete PROMOTE_TOKEN
```

Also delete stale `:testing`, `:stable-testing`, and `stable-testing-*` tags
from the GHCR package.

### 4. First Push

```bash
git add .
git commit -m "feat: initial customization"
git push origin main
```

### 5. Enable Renovate (Required)

- [ ] Create a **Classic PAT** (Settings → Developer settings → Personal access tokens → Tokens (classic))
  - Scopes: `repo` (full control) + `workflow` (update workflows)
- [ ] Add the token as repository secret **`RENOVATE_TOKEN`** (Settings → Secrets and variables → Actions)
- [ ] Enable **Settings → General → Pull Requests → Allow auto-merge**
- [ ] Configure branch protection for `main`:
  - Settings → Branches → Add rule
  - Set **Branch name pattern** to `main`
  - Enable "Require a pull request before merging"
  - Enable "Require status checks to pass before merging"
  - Add **`validate`** and **`build`** as required status checks
  - Enable "Require branches to be up to date before merging" (optional)
- [ ] Renovate will create a PR to pin your GitHub Actions to SHAs

Renovate targets `main`; merging a Renovate PR publishes the updated image.

**Agent skills:** `finpilot-onboarding` (branch protection), `finpilot-ci` (Renovate config)

### 6. Add "What Makes this Raptor Different" to README

- [ ] Open `README.md`
- [ ] Paste the raptor section template (see README or use the `finpilot-onboarding` skill)
- [ ] Fill in placeholders with your planned customizations
- [ ] Update the `*Last updated: [date]*` timestamp

**Agent skills:** `finpilot-onboarding` (raptor section), `finpilot-maintain` (maintenance requirement)

### 7. Participate in finpilot maintenance
- [ ] Use [finpilot issues](https://github.com/projectbluefin/finpilot/issues/new/choose)
  for reusable template or build-system improvements.
- [ ] Select the Clanker opt-in only on issues you create to send them to
  `3-clanker-queue`; maintainers may also apply that label.
- [ ] Port structural template changes to this repository through a focused PR.
  Renovate manages dependencies only; it does not synchronize arbitrary
  template files.

### 8. Deploy

Merge a PR to `main` to publish the production image, then deploy it:

```bash
sudo bootc switch --transport registry ghcr.io/YOUR_USERNAME/YOUR_REPO:stable
sudo systemctl reboot
```

## Optional: Production Features

### Enable Signing (Recommended)

This template uses keyless OIDC signing — no keys or secrets are required.

- [x] Uncomment the `Sign and publish` step in `.github/workflows/build-image.yml`
- [x] Confirm `id-token: write` permission is granted in the build job
- [x] Verify with `cosign verify` (see README "Optional: Enable Image Signing")

**Agent skill:** `finpilot-templates` (signing setup)

### Enable Rechunking (Optional)

- [ ] Edit `.github/workflows/build-image.yml`
- [ ] Set `ENABLE_RECHUNKING: "true"`
- [ ] Keep the default `RECHUNK_MAX_LAYERS: "128"` unless you have measured a reason to change it
- [ ] Confirm a publish build completes before deploying the new image

The current OCI-native chunkah action does not use `/usr/libexec/bootc-base-imagectl`. Package cadence classification is a separate advanced setup and is not required for basic rechunking.

**Agent skill:** `finpilot-ci` (rechunking compatibility and workflow setup)

## Agent Handoff Reference

Which skill to load for each checklist block above:

| Checklist step                        | Skill                                       |
| ------------------------------------- | ------------------------------------------- |
| Rename (step 1)                       | `finpilot-templates`, `finpilot-onboarding` |
| Enable Actions (step 2)               | `finpilot-onboarding`                       |
| Single-branch main (step 3)            | `finpilot-onboarding`, `finpilot-ci`        |
| Renovate + branch protection (step 5) | `finpilot-onboarding`, `finpilot-ci`        |
| Raptor section (step 6)               | `finpilot-onboarding`, `finpilot-maintain`  |
| Signing (optional)                    | `finpilot-templates`                        |
| Rechunking (optional)                 | `finpilot-ci`                               |

**Cross-link requirement**: Whenever you add or remove a package, app, or service **after** initial setup, update the README raptor section and its `*Last updated*` date. This is required by the `finpilot-maintain` skill.
