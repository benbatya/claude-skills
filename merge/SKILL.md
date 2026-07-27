---
name: merge
description: Land the current branch — or a whole stack of branches, bottom-up. Verify every branch is pushed and in sync with origin and that the local stack matches the upstream stack, find or create each branch's GitHub PR, run the repo's validation/acceptance checks plus each PR's CI, then merge bottom-up, retargeting each PR as its parent lands. Use when the user says "/merge", "merge this branch", "land this PR", "merge the stack", or "ship it".
---

# Merge the current branch, or a stack, via GitHub PRs

Take the checked-out branch from "work is done" to "landed on `main`": confirm the
remote has the work, get a PR for it, prove the change is good (local checks **and**
the PR's CI), then merge and tidy up.

If the branch is part of a **stack** (a chain of local branches each built on the one
below it), all of that applies to **every branch in the stack**, bottom-up: each one
is verified, each one gets a PR based on its stack parent, each one is validated, and
they land in ancestry order with each PR retargeted to `main` as its parent merges.

The text following the `/merge` invocation, if any, is an **option string** — see
"Options" at the end. A bare `/merge` uses the defaults.

**The first merge is the point of no return.** Steps 1–7 are all verification, and
any failure in them **stops the skill before anything has landed** — never merge past
a red check, never `--admin`/`--force` around a branch protection rule, and never
merge a PR whose tests you did not actually see pass. Because of that, the whole
stack is verified *before* the first branch merges (step 8), so a broken branch three
deep is caught while the tree is still untouched.

## Steps

1. **Identify the branch and refuse the obvious non-starters.**
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```
   - `HEAD` → detached, there is no branch to merge. **Stop and report.**
   - `main` → nothing to merge; `main` *is* the target. **Stop and report.**

   Call the result `CUR`, and remember it as the starting branch. Confirm `gh` is
   present and authenticated (`gh auth status`); if not, **stop** and tell the user
   to run `gh auth login` — do not fall back to a local `git merge` into `main`,
   which would bypass the PR and its checks entirely.

2. **Require a clean working tree.**
   ```bash
   git status --porcelain
   ```
   If this prints anything, there is uncommitted work. **Stop and report** — tell
   the user to commit it (e.g. `/push`) or stash it. Do **not** `git add -A` and
   commit on their behalf: this skill's job is to land reviewed work, and quietly
   sweeping the working tree into a PR that is about to merge is exactly the wrong
   moment for a surprise. Never stash, discard, or `git checkout -- .`.

   A clean tree is also a hard prerequisite for the rest of this skill, which checks
   out each branch in turn to validate it.

3. **Discover the stack and fix its order.** Same detection model as the `push` and
   `rebase` skills, so all three agree on what the stack is.
   ```bash
   git fetch origin --prune
   git for-each-ref --format='%(refname:short)' refs/heads
   ```
   Relative to `CUR`, for each local branch `B` (`B` != `CUR`, `B` != `main`):
   - **Below `CUR`** — `git merge-base --is-ancestor <B> <CUR>` succeeds.
   - **Above `CUR`** — `git merge-base --is-ancestor <CUR> <B>` succeeds.

   Drop any below-branch that has **already landed**
   (`git merge-base --is-ancestor <B> main` succeeds) — it is a stale local ref, not
   part of the work to merge; mention it in the final report.

   **The stack to land** is the remaining below-branches in ancestry order, followed
   by `CUR`: `main → B1 → B2 → … → CUR`. Order any set by ancestry — `X` comes before
   `Y` when `git merge-base --is-ancestor <X> <Y>` succeeds. A branch with nothing
   below it is a stack of one, and every step below still applies verbatim.

   **Branches above `CUR` are not landed by default.** They are still being built on;
   merging them because they happen to sit on top would ship unfinished work. List
   them in the final report as left behind, and note they will need `/rebase` onto
   the new `main` once the stack lands. (`/merge all` includes them — see Options.)

   If the ancestry forks — two below-branches where neither is an ancestor of the
   other — the "stack" is a tree and there is no single bottom-up order. **Stop and
   report** the shape; let the user say which line to land.

   Report the resolved order before doing anything else, so the user can see what is
   about to be landed and in what sequence.

4. **Verify every branch is pushed and in sync with `origin`.** Run this for **each**
   branch in the stack, bottom-up:
   ```bash
   git rev-list --left-right --count origin/<B>...<B>    # behind<TAB>ahead
   ```
   - **No `origin/<B>`** → never pushed. Push it: `git push -u origin <B>`.
   - **Ahead only** → fast-forward the remote: `git push -u origin <B>`. If the push
     is rejected, **stop and report** — never `--force` or `--force-with-lease`.
   - **Behind, or behind *and* ahead** → local and remote have diverged. **Stop and
     report** which branch; let the user reconcile (e.g. `/rebase`, or pull). Do not
     merge a PR whose head you have not seen.
   - **In sync** → continue.

   Collect the failures across the whole stack before stopping, so the user gets one
   complete picture rather than fixing branches one round-trip at a time.

5. **Verify the local stack matches the upstream stack.** Each branch being
   *individually* in sync is not enough: the remote refs must also be stacked in the
   same order, or merging bottom-up on the remote will not produce what the local
   ancestry implies. For each adjacent pair `P → B` in the stack (`P` = the parent,
   `main` for the bottom-most):
   ```bash
   git merge-base --is-ancestor origin/<P> origin/<B>     # must succeed
   ```
   If any pair fails, the remote branches are not stacked the way the local ones are
   — typically because one branch was force-pushed or rebased without the ones above
   it. **Stop and report** the pair that broke, and hand off to `/rebase`.

   Also confirm each branch's *own* commits are what you expect — the commits `B`
   adds over its parent:
   ```bash
   git log --oneline <P>..<B>
   ```
   An empty range means `B` adds nothing over its parent (an already-merged or
   duplicated branch). **Stop and report** rather than opening an empty PR.

6. **Find or create a PR for every branch, based on its stack parent.** For each
   branch `B` in the stack, with parent `P` (`main` for the bottom-most):
   ```bash
   gh pr view <B> --json number,title,url,state,isDraft,baseRefName,mergeable,mergeStateStatus,headRefOid
   ```
   - **A PR exists** → use it, after three checks:
     - `state` is `MERGED` or `CLOSED` → **stop and report** (already landed, or the
       user closed it deliberately).
     - `headRefOid` != `git rev-parse <B>` → the PR does not point at the commit you
       just verified. **Stop and report.**
     - `baseRefName` != `P` → the PR base does not match the local stack. This is the
       most common way a stack goes wrong: PR #2 based on `main` instead of on branch
       #1 shows a diff containing #1's changes and merges both at once. **Stop and
       report** the mismatch; retarget it with
       `gh pr edit <number> --base <P>` only if the user confirms that is the
       intended shape.

     If `isDraft` is true, mark it ready: `gh pr ready <number>`.
   - **No PR** → create one against the stack parent:
     ```bash
     gh pr create --base <P> --head <B> --title "<title>" --body "<body>"
     ```
     Write the title and body from that branch's own commits
     (`git log --oneline <P>..<B>`, `git diff --stat <P>...<B>`): a one-line title,
     then a short body covering what changed and how it was verified. For a stack,
     say in the body which branch it is based on and where it sits in the order, so a
     reviewer landing on PR #3 knows it is not meant to merge first. End the body
     with whatever trailer this environment mandates for PR bodies (e.g. the
     `🤖 Generated with [Claude Code]` line). If the repo has a PR template, fill it
     in rather than ignoring it.

7. **Validate the whole stack — this is the gate.** Two independent sources per
   branch, and **both** must be clean for **every** branch before *any* branch
   merges. Validating up front costs a few checkouts and buys the guarantee that the
   skill never lands half a stack because the top branch was broken.

   For each branch `B` in the stack, bottom-up: `git checkout <B>`, then:

   **7a — Local checks.** Discover what this repo actually runs; do not assume. Look
   at `CLAUDE.md`/`AGENTS.md` (they often name the exact commands), then
   `package.json` `scripts`, then a `Makefile`, `justfile`, `pyproject.toml`,
   `Cargo.toml`, etc. Run the validation-shaped ones — typically typecheck, lint,
   test, and any project-specific acceptance check (e.g. `npm run check:reader`) —
   plus a build if the project has one. Run them from the directory that owns the
   manifest, and re-discover per branch: a branch may add or rename a check.

   Report each command and its exit status, labelled with the branch it ran on. **Any
   non-zero exit stops the skill**: report the branch, the failing command, and its
   output, and do not merge anything. If the repo has no discoverable checks at all,
   say so explicitly in the final report rather than implying the stack was validated.

   **7b — PR checks (CI).** Wait for that branch's PR checks to finish:
   ```bash
   gh pr checks <number> --watch --fail-fast
   ```
   - **All pass** → continue to the next branch.
   - **Any fail** → **stop and report** which branch, which check, with a link. Never
     merge over a failing check and never bypass it with `--admin`.
   - **No checks configured** → `gh pr checks` exits non-zero with "no checks reported
     on the ... branch". That is *not* a failure; note in the report that the PR runs
     no CI, so 7a was the only validation.
   - Long-running CI: keep waiting rather than polling in a tight loop. If it is still
     running after a long stretch, report the in-progress state and let the user
     decide — do not merge a PR whose checks have not settled.

   Also confirm review state is satisfied for each PR — if the repo requires approvals
   and a PR has none (or has a `CHANGES_REQUESTED` review), **stop and report**; do
   not try to merge and let the API reject it.

8. **Merge bottom-up, retargeting as you go.** Only now does anything become
   irreversible. For each branch `B` in the stack, in ancestry order:

   **8a — Retarget.** If `B` is not the bottom-most, its PR is still based on a branch
   that has now been merged and deleted. GitHub retargets such PRs automatically, but
   it is asynchronous and not guaranteed, so do it explicitly and verify:
   ```bash
   gh pr edit <number> --base main
   ```

   **8b — Confirm mergeability.** GitHub recomputes this after a retarget, so re-read
   it rather than trusting the value from step 6:
   ```bash
   gh pr view <number> --json mergeable,mergeStateStatus
   ```
   - `mergeable: CONFLICTING` (or `mergeStateStatus: DIRTY`) → conflicts with the
     base. **Stop and report**; tell the user to `/rebase`. Never resolve conflicts as
     part of a merge action.
   - `mergeStateStatus: BEHIND` → the base moved and the repo requires the branch be
     up to date. **Stop and report**, suggesting `/rebase` (then re-run `/merge`, so
     the checks run against what actually lands).
   - `BLOCKED` → a protection rule is unsatisfied (missing review, required check not
     run). **Stop and report** which one. Do not bypass it.
   - `CLEAN` / `UNSTABLE`-with-only-optional-checks-passing → merge.

   **8c — Merge.**
   ```bash
   gh pr merge <number> --merge --delete-branch
   ```
   Use the merge method the repo's history shows (see Options); `--merge` is the
   default here, and for a multi-branch stack it is strongly preferred — see the
   squash warning in Options.

   **8d — Advance local `main`** so the next branch's mergeability and diff are
   computed against what actually landed. This refspec form fast-forwards `main`
   without checking it out, so the working tree is untouched:
   ```bash
   git fetch origin main:main
   ```

   **If a merge fails part-way through the stack**, stop immediately and report
   precisely which branches landed, which did not, and that the remainder now sits on
   an older `main` and needs `/rebase` before `/merge` can be re-run. Do not attempt
   to continue past a failure — every branch above it is now built on a base that no
   longer matches.

9. **Return to a clean state.**
   ```bash
   git checkout main
   git pull --ff-only origin main
   ```
   Then delete each landed branch locally with the **safe** delete, which refuses if
   the commits are not actually reachable from `main`:
   ```bash
   git branch -d <B>
   ```
   If `git branch -d` refuses, leave that branch alone and say so — never `-D`. (A
   squash or rebase merge rewrites the commits, so `-d` may legitimately refuse;
   report it as expected in that case rather than forcing.)

   If any branch in the stack did **not** land, or there are branches above `CUR`,
   end on the lowest surviving branch instead of `main` so the user is positioned to
   continue.

10. **Report**, per branch and in order: the PR number and URL, each local check and
    its result, the CI outcome (or that there was none), and the merge method used.
    Then the overall picture: `main`'s new position, which branches were deleted
    locally and remotely, which were left behind (above-branches, stale already-landed
    refs) and what they need next. If the skill stopped early, say exactly which
    branch and which step stopped it, what has already landed, and what the user
    should do next.

## Options

The text after `/merge` selects the merge method and adjusts scope:

- `squash` → `gh pr merge --squash` · `rebase` → `gh pr merge --rebase` ·
  default (or `merge`) → `gh pr merge --merge`.

  **For a stack of more than one branch, squash and rebase are not safe to loop.**
  Both rewrite the branch's commits, so what lands on `main` is a *different* commit
  from the one the next branch is built on. The next PR's diff then re-contains its
  parent's changes and will usually conflict. If the user asks for either on a
  multi-branch stack: say so, merge **only the bottom-most** branch, then stop and
  tell them to `/rebase` the remainder onto the new `main` and re-run `/merge`. A
  merge commit (`--merge`) puts the parent's exact commits on `main`, which is why it
  is the default and the only method that loops cleanly.
- `all` → also land the branches **above** `CUR`, extending the stack upward in
  ancestry order. Only use this when the user explicitly asks for it; the default
  deliberately stops at `CUR`.
- `no-verify` / `skip-checks` → skip **7a** (local checks) only, on every branch. CI
  in 7b is still required and is still a hard gate. Call this out prominently in the
  report.
- A bare number (e.g. `/merge 12`) → operate on that PR alone, ignoring stack
  detection. If its head branch is not checked out, `gh pr checkout 12` first (which
  requires a clean tree per step 2).

If the option string is anything else, treat it as intent to describe the merge (or
ask), not as a flag to pass through to `gh`.

## Notes

- **Never merge on unverified evidence.** If CI did not run and no local checks exist,
  say so plainly in the report — an unvalidated merge must never be described as a
  validated one.
- **Verify everything, then merge everything.** Steps 3–7 mutate nothing but the
  remote refs of branches that were already meant to be pushed. Keep it that way: no
  merging until the entire stack is green.
- Never `--admin`, `--force`, `--force-with-lease`, or `-D`. Never resolve conflicts,
  and never `git merge` into `main` locally as a workaround for a blocked PR — the PR
  *is* the gate.
- Never push `main` directly; it advances only through the merges in step 8 and the
  fast-forwards in 8d/9. This matches the `push` skill, which refuses `main`.
- This skill does not rebase. If a PR is behind, conflicting, or the remote stack has
  drifted out of order, it stops and hands off to `/rebase`, so the validation in
  step 7 always reflects the code that actually lands.
- Companion skills: `/change` starts the branch, `/push` publishes it (stack-aware),
  `/rebase` keeps it current (stack-aware), `/merge` lands it (stack-aware).
