# Upstream Deviations Register

A maintained, fork-local record of every deliberate change this fork makes
relative to upstream `https://github.com/projectbluefin/finpilot`.

Every change listed here is an **exception**. The fork's goal is to be
byte-identical to upstream. Each exception must be **removed the moment
upstream fixes the underlying bug** (or, for identity/content, is never
removed because it *is* the fork). Nothing here is a blessed permanent
divergence beyond the fork's own identity.

## Fork Metadata

- **Upstream repo:** `https://github.com/projectbluefin/finpilot`
- **Upstream remote alias:** `upstream`
- **Fork created from upstream commit:**
  `0d02b8ec2da31961241417413cc6f5989cec0402`

## Catch-up Pointer

- **Upstream commit last fully scanned:**
  `be4b0ebe7b493a18a20d27539b4e3eba23f093c1` (2026-09-18)
- **Latest upstream `main` HEAD observed:**
  `be4b0ebe7b493a18a20d27539b4e3eba23f093c1` (2026-09-18)

Fork point (creation) is `0d02b8ec2da31961241417413cc6f5989cec0402`. **Before
relying on any claim here, re-fetch upstream (`git fetch upstream main`) and
re-verify each entry against upstream's current content.**

## Catch-up Notes (2026-09-18)

Adopted roughly 150 upstream commits since the previous pointer `a69e771`:

- **Removed** the `iso/rclone/` subsystem and its `.dockerignore` entry
  (unrecorded deviation — upstream dropped it).
- **Restored static Brewfile validation** (`.pre-commit-config.yaml`,
  `validate-brewfiles.yml`, new `build/validate-brewfiles.sh`); the fork's
  inline `brew bundle check` was left over from before upstream's #288/#323
  security refactor.
- **Dropped `secrets: inherit`** from `label-enforcement.yml` (upstream #318).
- **Restored `set -euo pipefail`** in `install-jetbrains-toolbox`
  (upstream #315 checksum gate).
- **Adopted** new files: `.shellcheck-scope`, `build/validate-flatpaks.sh`,
  `approve-trusted-promotion-runs.yml`, `unit-tests.yml`, `tests/unit/*.bats`.
- **Adopted upstream's `clean.yml`** which derives the cleanup package name
  from the repository name — this makes the fork's former hardcoded
  `packages: osprey-2-stage` row obsolete (exception removed).
- **Adopted upstream's rewritten `validate-brewfiles.yml`** (digest exception
  removed; workflow no longer differs).
- **Adopted upstream's `renovate.json`**, including removal of the
  human-review rule for GitHub Actions updates (per upstream's decision;
  automerges action digest bumps). Not an exception.
- Containerfile digest pins re-aligned to upstream (Renovate churn is expected
  to drift again — digest pins are never recorded as differing until verified).
- Adopted upstream's `AGENTS.md` (re-adds the #281 release-gate "Known gap"
  paragraph) and `.agents/skills/finpilot-templates/SKILL.md` (upstream's
  worked-example wording). These are no longer deviations.

## State of This Register

This register reflects **`origin/main` — the fork's committed remote `main`** —
compared against `upstream/main`. All fork work is merged to `main`, so the
register documents the deliberate differences that live on the fork's `main`
branch. The committed tree is byte-identical to `upstream/main` except for the
exceptions in the table below. Deviations previously present (a
`branches: [main]` restriction on two validation workflows, and a large local
promotion workflow) have been **reverted to upstream** and are recorded in
Appendix: History for accountability only — they are not current exceptions.

## Rules (non-negotiable)

1. **Upstream only changes upstream.** This fork never edits, patches, or
   proposes fixes to `projectbluefin/*` or `ublue-os/*`. Do not open issues or
   PRs against upstream, and do not suggest upstream fixes.
2. **Verify, never assume.** Every entry must be grounded by a live
   `git diff`/`git show` against current upstream, not by memory or by stale
   comments. Re-fetch before trusting the catch-up pointer.
3. **A prior session** deviated by guessing, by trusting its own notes and
   comments instead of verifying, and by failing to demonstrate the reason for
   each deviation — then fell back to asking for forgiveness under pressure.
   Recording every exception with a verifiable justification is the
   correction. When in doubt, adopt upstream.
4. **Act on the known rule; do not ask the obvious question.** When the
   correct action is already determined by these rules, take it and show the
   evidence. Do not ask the user for direction (or for forgiveness) in lieu of
   doing the verified work. Demonstrate, then confirm.
5. **Byte-identical is the resting state.** Except for the exceptions below,
   every file matches upstream exactly.

## Exception Register (current, `origin/main`)

| File | Difference (vs upstream) | Class | Removal trigger |
|------|--------------------------|-------|-----------------|
| `Containerfile` | `Name: osprey-2-stage`; `IMAGE_NAME=osprey-2-stage`; `IMAGE_VENDOR=dhodyn` | Identity | Never |
| `Justfile` | `IMAGE_NAME := "osprey-2-stage"` | Identity | Never |
| `README.md` | Title, raptor section, cosign URLs, checklist state | Identity | Never |
| `artifacthub-repo.yml` | `repositoryID: osprey-2-stage` | Identity | Never |
| `iso/iso.toml` | `bootc switch ... ghcr.io/dhodyn/osprey-2-stage:stable` | Identity | Never |
| `custom/ujust/README.md` | `localhost/osprey-2-stage:stable` | Identity | Never |
| `custom/flatpaks/default.preinstall` | Active Spotify + Thunderbird | Intended fork content | Never (intended) |
| `custom/brew/default.Brewfile` | Active `neovim` + `helix` | Intended fork content | Never (intended) |

> **Verification note:** every row here must be re-checked against current
> upstream before each use. Re-fetch and `git diff` before relying on it.
> The two previous rows for `clean.yml` and `validate-brewfiles.yml` were
> removed in the 2026-09-18 catch-up because those files are now
> byte-identical to upstream.

## Fork-local files (not exceptions)

These exist only in the fork, so `git diff upstream/main origin/main` lists
them but they are **not** deviations and have no removal trigger:

- `UPSTREAM_DEVIATIONS.md` — this register.
- `.agents/skills/finpilot-align-upstream/SKILL.md` — fork-local align skill
  (upstream ships no such file).

## Appendix: History (reverted, not current)

For accountability, the deviations created in a prior session and since
reverted to upstream are recorded here. They are **not** current exceptions
and must not be re-applied:

- `.github/workflows/pr-validation.yml` — was restricted to
  `branches: [main]`; now byte-identical to upstream (`[main, stable]`).
- `.github/workflows/label-enforcement.yml` — was filtered to
  `branches: [main]`; now byte-identical to upstream.
- `.github/workflows/promote-main-to-stable.yml` — was a ~450-line local
  implementation; now byte-identical to upstream's thin caller.
- Various `.agents/skills/*`, `AGENTS.md`, `.github/SETUP_CHECKLIST.md`,
  `custom/flatpaks/README.md` — were edited to assert fabricated deviations
  (e.g. "signing disabled by default", a wrong flatpak path, a replaced
  brewfiles-CI claim); all now byte-identical to upstream.

## Process Each Session

1. `git fetch upstream main`
2. Read this register; note the catch-up pointer.
3. Scan from the catch-up pointer, not from the fork origin, for upstream
   changes: `git log --oneline <CATCH_UP_POINTER>..upstream/main`
4. Adopt upstream's content for any exception whose removal trigger has
   landed; update the register.
5. Leave the register truthful to the **`origin/main` vs `upstream/main`**
   state, with each surviving exception verified and explained.
