---
name: finpilot-align-upstream
description: Align the fork to upstream projectbluefin/finpilot. Use at the start of every session and whenever touching a file. Reminds the agent to scan upstream from the recorded fork commit, keep the fork byte-identical, maintain UPSTREAM_DEVIATIONS.md (fork-local), and never change or suggest fixes upstream.
---

# Align to Upstream

## Purpose

This fork's resting state is **byte-identical to upstream**
`projectbluefin/finpilot`. Any deviation is a temporary exception, recorded
and justified in `UPSTREAM_DEVIATIONS.md`, to be removed once upstream fixes
the underlying bug. The goal is to keep catching up to upstream, never to
drift further.

## Hard Rules

1. **Never change or suggest a fix upstream.** This fork does not edit,
   patch, open issues against, or propose changes to `projectbluefin/*` or
   `ublue-os/*`. Upstream is not ours to change.
2. **Verify before trusting.** Do not rely on memory, notes, or old comments.
   Re-fetch upstream and `git diff`/`git show` the actual files before
   asserting any deviation or claim.
3. **Exceptions are temporary and recorded.** Every divergence must have a
   verified justification and a removal trigger in `UPSTREAM_DEVIATIONS.md`.
4. **Keep the register current.** On every session update the catch-up pointer
   and the exception table to match the real, verified state.
5. **Byte-identical is the default.** If there is no verified reason to
   differ, the file must match upstream exactly.
6. **Act on the known rule; do not ask what to do in the obvious case.** When
   the correct action is already determined by these rules, take it and show
   the evidence. Do not ask the user for direction (or for forgiveness) in
   lieu of doing the verified work. Hiding behind a question when the answer
   is already known is a way of dodging the task. Demonstrate, then confirm.

## Session Start

1. `git fetch upstream main`
2. Read `UPSTREAM_DEVIATIONS.md` (fork-local, maintained register).
3. Note the catch-up pointer (last upstream commit scanned).
4. Scan `0d02b8ec..upstream/main` for upstream changes.
5. Adopt upstream's content for any exception whose removal trigger has landed;
   update the register.

## How to Compare (the one correct command)

To list every file that differs between the **working tree** and upstream, use:

```sh
git diff --name-only upstream/main
```

That is the authoritative comparison. Do **NOT** write ad-hoc comparison loops
or use `git diff --no-index` with `<(...)`/`git show upstream/main:FILE`
process substitutions — those treat the upstream blob path as a literal local
filename, so every file spuriously "differs" and you will misreport the true
diff count (this happened once: a simple "what differs" question produced a
false "26 files / 16 reverted" panic). One correct command beats many ad-hoc
ones. If a result contradicts your earlier findings, stop and rerun the single
authoritative command; do not pile on more experimental checks.

## Ground Truth Anchors

- Upstream repo: `https://github.com/projectbluefin/finpilot` (remote `upstream`)
- Fork created from upstream commit:
  `0d02b8ec2da31961241417413cc6f5989cec0402`
- Register file: `UPSTREAM_DEVIATIONS.md`

## Self-Check

- [ ] Did I re-fetch upstream and verify against current `upstream/main`,
      not memory?
- [ ] Is every surviving deviation recorded with justification + removal
      trigger in `UPSTREAM_DEVIATIONS.md`?
- [ ] Have I adopted upstream for anything whose fix has landed?
- [ ] Did I refrain from changing or suggesting anything upstream?
- [ ] Did I act on the known rule and demonstrate the evidence instead of
      asking the obvious question or dodging with a question?
- [ ] Did I use `git diff upstream/main --` (the one correct command) and not
      an ad-hoc `--no-index`/`<(git show)` comparison loop?
