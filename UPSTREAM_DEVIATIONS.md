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
  `341cbac4a179f67dd8b965f73c4b1f6a75cf9b86` (2026-09-20)
- **Latest upstream `main` HEAD observed:**
  `341cbac4a179f67dd8b965f73c4b1f6a75cf9b86` (2026-09-20)

Fork point (creation) is `0d02b8ec2da31961241417413cc6f5989cec0402`; the
scan range is now fully caught up to `341cbac…`. **Before relying on any
claim here, re-fetch upstream (`git fetch upstream main`) and re-verify each
entry against upstream's current content.**

## Catch-up Notes (2026-09-21)

Cumulative catch-up from `a69e771` (the fork's last recorded pointer on
`main`) through `341cbac`, incorporating the earlier `be4b0eb` catch-up that
lived on unmerged branch `align/upstream-2026-09-18` (#196) and was superseded
by this one. Highlights:

- **Skills restructured back to generic names.** Upstream replaced the
  `finpilot-*` skill tree with plain `build`/`ci`/`customize`/`onboarding`/
  `overview`/`troubleshooting` skills, deleted `skill-improvement` and
  `docs/skills/*`, and rewrote AGENTS.md. The fork's
  `.agents/skills/finpilot-align-upstream/SKILL.md` remains fork-local.
- **The two-branch release model moved to digest promotion.** Upstream deleted
  the `approve-trusted-promotion-runs.yml` gate and `label-enforcement.yml`;
  `promote-main-to-stable.yml` now just opens a squash PR gated on a tree-match
  to `main`, and the new `execute-release.yml` promotes the exact digest `main`
  built — `stable` never rebuilds. `clean.yml` derives the cleanup package name
  from the repository name (the fork's former `packages:` row is obsolete).
- **Build phases renamed.** `build/10-build.sh`/`clean-stage.sh` became
  `build/10-overlay.sh`, `build/20-packages-and-services.sh`,
  `build/90-cleanup.sh`; examples renumbered (tailscale, gnome-extensions,
  nvidia, desktop-swap); `custom/config/` and `custom/files/` added; the
  `fonts.Brewfile`, jetbrains, benchmark, and comic-mono files dropped.
- **Containerfile restructured.** Image metadata moved into the Containerfile
  as ARGs; `BASE_IMAGE_NAME` has no default and is derived from the `FROM` line;
  `build/00-image-info.sh` requires `IMAGE_NAME`/`IMAGE_VENDOR`/`UBLUE_IMAGE_TAG`.
- **Justfile rewritten.** VM/QCOW2 recipes, `_format-justfiles`, and a
  contract/template test split (`just test-contract`/`test-template`).
- **Tests split into `tests/contract` + `tests/template`** replacing
  `tests/unit`, including a new `identity_test.bats` that fails when the
  Containerfile `# Name:`/`ARG IMAGE_NAME`, Justfile `IMAGE_NAME` default, and
  artifacthub `repositoryID` disagree.
- **`iso.toml` no longer carries an image reference** — the `bootc switch`
  line was removed upstream; image identity flows from image-info.json via the
  Justfile. The fork's old `iso.toml` exception is resolved.
- **`custom/ujust/README.md` no longer carries an image reference** — the
  fork's old exception is resolved.
- **`.hadolint.yaml` renamed to `.github/hadolint.yaml`**; `.shellcheck-scope`
  deleted (shellcheck now walks every tracked `*.sh`); `validate-flatpaks.yml`
  pins `ubuntu-24.04` and refreshes apt before installing flatpak; `renovate.json`
  automerges everything short of major.

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
| `Containerfile` | `# Name: osprey-2-stage`; `ARG IMAGE_NAME="osprey-2-stage"`; `ARG IMAGE_VENDOR="dhodyn"` | Identity | Never |
| `Justfile` | `IMAGE_NAME := env("IMAGE_NAME", "osprey-2-stage")` | Identity | Never |
| `README.md` | Title, raptor section, cosign URLs, bootc switch examples | Identity | Never |
| `artifacthub-repo.yml` | `repositoryID: osprey-2-stage` | Identity | Never |
| `custom/flatpaks/default.preinstall` | Active Spotify + Thunderbird (upstream ships Thunderbird + Flatseal + ExtensionManager) | Intended fork content | Never (intended) |
| `custom/brew/default.Brewfile` | Active `neovim` + `helix` (upstream ships assertions only) | Intended fork content | Never (intended) |

> **Verification note:** every row here must be re-checked against current
> upstream before each use. Re-fetch and `git diff` before relying on it.
> Rows previously recorded for `iso/iso.toml`, `custom/ujust/README.md`,
> `.github/workflows/clean.yml`, and `.github/workflows/validate-brewfiles.yml`
> were removed in the 2026-09-21 catch-up because those files are now
> byte-identical to upstream.

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
