# Plan: Single-Branch `main` Model for osprey

## Goal

Replace the two-branch model (`main` → `:stable-testing`, `stable` → `:stable` + promotion PR) with a **single `main` branch** where `main` *is* production. All gatekeeping happens on PRs; merging publishes `:stable`. Kill the promotion/sync/gate machinery (~500 lines), the `PROMOTE_TOKEN` PAT, the `stable` ruleset, and the `:stable-testing`/`:testing` tag stream.

## Final architecture

```
PR ──(required checks: validate + build)──> merge (rebase/merge/squash) into main ──> publish :stable + stable-daily-* + version tags
```

- One branch: `main`. No `stable`, no promotion, no second ruleset, no PAT.
- PRs build the image **and write the layer + DNF cache**; the post-merge `main` build reuses that cache → no recompilation, just download/assemble/sign/push.
- Merge method is **context-driven**: repo allows all three (merge commits, rebase, squash). Renovate keeps its `squash` automerge strategy — no change to `renovate.json`.
- **No forks**: public personal repos can't disable forking, so fork PRs are blocked by a workflow guard that fails the `build` check.

## Confirmed decisions

- PRs run a full image build; `build` is a required check on `main` (plus `validate` from `pr-validation.yml`).
- `main` rebuilds after merge but is fast via cache reuse — PRs write `REGISTRY_CACHE_WRITE=1` and the DNF cache (no poisoning risk since no forks).
- A 2-commit PR (feat + fix) rebase-merged lands as **2 commits** on `main`.
- Base image changes ~daily via Renovate digest PRs (every 6h schedule); those build cold **once** (on the PR), and their merge reuses the PR's cache on `main`.
- The destructive GitHub-side steps are run **last**, individually reversible.

## 1. Workflow changes

### `build-image.yml` (rewrite)

- `on.push.branches: [main]` + add `on.pull_request.branches: [main]` (same `paths-ignore`: `**.md`, `.github/workflows/*validate*.yml`).
- Delete `PRODUCTION_BRANCH` env and the `Determine image tag` step; hardcode `TAG_STREAM: "stable"`.
- `REGISTRY_CACHE_WRITE: "1"` (was `1` only for push builds).
- `Save DNF cache`: `allow-write: true`, drop the `event_name != 'pull_request'` guard.
- New guard step (fails fork PRs), runs only on `pull_request` when `github.event.pull_request.head.repo.full_name != github.repository` → `exit 1`.
- `Finalize branch tags`: strip the `testing`/`stable-testing` branch; always keep generated tags + `:stable`.
- Publish/sign steps stay guarded by `github.event_name != 'pull_request'`.
- Rewrite the header comment for the single-branch model.

### Deletions & tweaks

- Delete `promote-main-to-stable.yml` and `sync-stable-to-main.yml`.
- `pr-validation.yml`: branches → `main` only; drop `merge_group`.
- No change to `renovate.json`, `Justfile`, `clean.yml`, `validate-*.yml`, `label-enforcement.yml`.

## 2. Repo settings — commands to run (run these LAST)

```bash
gh api -X DELETE repos/dhodyn/osprey/git/refs/heads/stable
gh api -X DELETE repos/dhodyn/osprey/git/refs/heads/auto/promote-main-to-stable
gh api -X DELETE repos/dhodyn/osprey/rulesets/20697458          # stable release protection
gh secret delete PROMOTE_TOKEN
```

- `main` protection (Settings → Branches): require PR, no direct pushes, required checks **`validate`** + **`build`**; "require branches up to date" optional.
- Settings → Merge button: allow merge commits + rebase + squash.
- Keep "Allow auto-merge" and "Allow GitHub Actions to create/approve PRs" (Renovate needs both).

## 3. Docs & skills updates

- `AGENTS.md` — rewrite `## Branch Strategy` + `## Release Workflow` for single-branch/PR-gated/main→`:stable`; delete PAT/merge-queue/ruleset/PROMOTE_TOKEN lore; fix critical rules #7/#8/#9 and the tag table; bump `Last Updated`.
- `README.md` — Phase 3, Build System bullets, Development Workflow, deploy examples (`:stable` only).
- `.github/SETUP_CHECKLIST.md` — single-branch setup; remove `PROMOTE_TOKEN`; add the gh commands above; `bootc switch ... :stable`.
- Skills: `finpilot-ci` (workflow map + replace "Branch Promotion and Tags" with the single-branch cache model), `finpilot-onboarding` (PR + checks `validate`/`build`, fork guard, merge methods), `finpilot-maintain`, `finpilot-overview`, `finpilot-troubleshooting` (generalize the promotion-gate `startup_failure` permissions note).

## 4. GHCR cleanup (optional)

Delete stale `:testing`, `:stable-testing`, `stable-testing-*` tags manually, or let `clean.yml` age them out (keeps last 7).

## 5. Testing before touching the real repo (Options 1 + 2)

The real-repo code changes are non-destructive and land as a PR; destructive steps are last.

### Phase C — cache benchmark FIRST (scratch clone, ~40–60 min)

Run on the **pristine, untouched repo** (clone is from committed HEAD; no working-tree edits yet). The benchmark only exercises `Justfile`/`Containerfile`/`build/`, which Phase A does not change, so nothing about the workflow edits matters here — this is the riskiest-assumption test and must pass before any file changes.

1. Clone current repo → `/tmp/opencode/osprey-bench`
2. `buildah prune --force` — clears build cache + intermediate images (base images kept; verified: buildah 1.43.2, rootless storage). **This is the only approved destructive action pre-gate.**
3. **Cold build**: `time IMAGE_NAME=osprey-bench IMAGE_VENDOR=localonly just build` (fully cold; never touches `ghcr.io/dhodyn/osprey`'s real cache)
4. Add powertop to `build/10-build.sh` (scratch clone only)
5. **Warm build**: same command, timed
6. Compare durations → cleanup scratch clone

### Phase A — implement workflow changes (Section 1)

Local working-tree edits only: no commit, no push until the gate passes.

### Phase B — static validation (seconds)

- `actionlint .github/workflows/*.yml`
- YAML parse modified workflows (`python3 -c "import yaml; yaml.safe_load(open('file.yml'))"`)
- `just --list`
- `shellcheck build/*.sh`
- grep for stray `stable`/`promote`/`testing`/`PROMOTE_TOKEN`

### Gate

If warm ≪ cold (expected: 20+ min → a few min), the design holds → proceed with docs/skills + migration. If not, redesign before shipping.

## 6. Migration ordering (safety)

1. Benchmark (Phase C) on the pristine repo — no working-tree changes before this passes.
2. Implement + validate workflows (Phase A/B).
3. Docs/skills PR.
4. Merge to `main`, confirm `:stable` publishes.
5. Run destructive gh commands last.

## Research: upstream base image tag scheme (quay.io/fedora-ostree-desktops/silverblue)

Verified 2026-08-12 via `skopeo list-tags`. 227 non-sig tags total:

- **Floating major aliases**: `41`, `43`, `44`, `45`, `rawhide`
- **Daily builds**: `44.YYYYMMDD.N` (e.g. `44.20260812.0`), each with `-x86_64`/`-aarch64` variants
- **Cosign signature tags**: `sha256-<digest>.sig` (thousands of them — always filter `rg -v '\.sig$'` when listing)
- `rawhide` also gets daily builds (`rawhide.YYYYMMDD.N`)

Implication: our pinned `FROM ...silverblue:44@sha256:e59d4b88...` is a **moving tag** (daily churn), which is why Renovate tracks its digest every 6h. When manually checking for base updates, resolve against the daily tags or the `:44` alias digest — not the sig tags.

## Environment notes

- Tools available: `actionlint`, `shellcheck`, `just`, `podman` 5.8.4, `skopeo`, `buildah` 1.43.2, `python3`, `jq`.
- Origin: `https://github.com/dhodyn/osprey.git`; local branches `main` + `stable`.
