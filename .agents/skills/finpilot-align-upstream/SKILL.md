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
7. **Do NOT keep a permanent `upstream` git remote.** Add it only to compare
   with upstream, then remove it. A persistent `upstream` (name or URL to
   `projectbluefin/finpilot`) makes `gh` resolve the base repository to
   **upstream** instead of the fork, so `gh pr create --base main` fails with
   `No commits between projectbluefin:main and ...`. `gh` keys off the git
   remotes it sees, so never leave upstream installed when creating/merging
   PRs. Sequence: add upstream → fetch/compare → remove upstream → then do any
   git/gh work against the fork.

## Session Start

1. Add the upstream remote transiently: `git remote add upstream
   https://github.com/projectbluefin/finpilot.git` (if not already present).
2. `git fetch upstream main`
3. Read `UPSTREAM_DEVIATIONS.md` (fork-local, maintained register).
4. Note the catch-up pointer (last upstream commit fully scanned) and its
   value — do not re-scan from the fork point, only from this pointer.
5. Scan from the catch-up pointer, not from the fork origin. Use the current
   catch-up value from `UPSTREAM_DEVIATIONS.md`:
   `git log --oneline <CATCH_UP_POINTER>..upstream/main`
   (e.g. if the pointer is `abc123`, run `git log --oneline abc123..upstream/main`,
   **not** `0d02b8ec..upstream/main` every time).
6. Adopt upstream's content for any exception whose removal trigger has landed;
   update the register.
7. **Remove the transient remote holds nothing more is needed** —
   `git remote remove upstream` — BEFORE any `git push`, `gh pr create`,
   `gh pr merge`, or branch-retry, so `gh` resolves the fork, not upstream.

## Bug Report / Troubleshooting in a Fresh Session

When a user reports a problem or bug, do NOT guess a fix or default to
"change whatever seems wrong." The fork must stay byte-identical to upstream
except for recorded exceptions. Procedure:

1. **Orient from persisted state, not memory.** Read `UPSTREAM_DEVIATIONS.md`
   — fork point, catch-up pointer, exception register.
2. **Find the file involved.** Identify which path the bug touches.
3. **Is that file a recorded exception?** Check the register table. If the file
   is not in the register, it must be byte-identical to upstream, so the bug is
   **upstream's** bug, not a fork deviation.
4. **Verify against live upstream** (`git diff upstream/main -- <file>`, the
   one correct command). If it is byte-identical, do NOT patch it in the fork:
   that would silently adopt a change upstream does not have. Report to the user
   that the behavior comes from upstream, not from this fork.
5. **If the file IS a recorded exception:** fix it **within the exception's
   justification only** — never expand a deviation; keep the change minimal and
   consistent with the recorded removal trigger.
6. **If no exception covers it and the user still needs a fix:** the correct
   answer per rule 1 is to **not** modify upstream-owned content. The fork
   waits for upstream; this is not a fork bug to patch. Do not invent a workaround
   that diverges from upstream.

Never let a user-reported bug be an excuse to break byte-identical alignment
into an un-recorded deviation.

## Answering PR / Run Questions (thorough on the FIRST pass)

When asked "will this merge?", "is this auto-merging?", or "what is this run?",
do the complete investigation in one go — never answer from partial data.
Incomplete answers cause the user to re-ask with the evidence you missed.

**Gather ALL of these in parallel the first time:**
1. `gh pr view <n> --repo <fork> --json body,labels,reviewRequests,mergeStateStatus,mergeable,autoMergeRequest,state`
2. `gh pr checks <n> --repo <fork>` — full check list with states
3. `gh api .../branches/stable/protection` — required checks, required reviews,
   merge-queue, rulesets
4. The originating workflow run (promote/build) — `gh run view <id> --log` for
   the actual step output, NOT just its green/red conclusion
5. `gh api .../pulls/<n>/requested_reviewers`

**Cross-check the PR BODY against facts.** The promotion PR body is generated
from upstream shared templates (`render_pr_body.py`, `render_gate_section.py`)
and may contain boilerplate that does NOT match this fork: references to
`@projectbluefin/maintainers`, "2 approvals", "Auto-merge scheduled for
Tuesday/Thursday 04:00 UTC (bluefin/dakota/bluefin-lts)", or desktop-screenshot
URLs pointing at `projectbluefin.github.io`. Verify each claim against the
fork's own branch protection, reviewers (empty), auto-merge state (null), and
required checks. Report the mismatch explicitly rather than repeating upstream
boilerplate as if it were the fork's true state.

**Never merge-opinion without evidence:** state what actually happens
(required check `validate` passes; `enforce` parked-run is not required),
what does NOT happen (auto-merge is not enabled; no approvals required), and
what the user must do (merge manually with squash).

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
- [ ] For a reported bug: did I check whether the file is a recorded exception,
      verify it against upstream, and avoid un-recorded divergence? (Bug Report
      section)
- [ ] For a PR/run question: did I gather body, checks, protection, reviewers,
      and workflow logs ALL at once, and cross-check any boilerplate in the PR
      body against fork facts before answering? (Answering PR/Run Questions)
