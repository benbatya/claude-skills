---
name: push
description: Create or update the branch's plans/<branch_name>.md with the changes and decisions it made, commit any uncommitted work, then push the branch to origin, setting upstream tracking if it is not yet configured. Never pushes the main branch. If the current branch is part of a stack of local branches, push every branch in the stack, skipping any that has already landed in main. Use when the user asks to push the current branch, "push this branch", "push my work", or "push the stack".
model: sonnet
---

# Push the current branch (stack-aware)

Push the checked-out git branch to the `origin` remote. Before pushing, bring the
branch's `plans/<branch_name>.md` up to date with the changes and decisions it has
made — creating it if the branch does not have one yet — then commit it along with
any other uncommitted changes, so nothing in the working tree is left behind and
the branch's reasoning is recorded. If the current branch is part of a **stack** (a
chain of local branches each built on the one below it), push every branch in the
stack so all their remotes advance together.

## Steps

1. Determine the current branch:
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```
   If this returns `HEAD`, the repo is in a detached-HEAD state — stop and tell
   the user there is no branch to push (and nothing to commit onto).

   **If the current branch is `main`, stop and report** — this skill never
   pushes `main` to the remote. Do not commit or push anything; `main` advances
   only through merges/PRs, not a direct push.

2. **Write the branch's plan file, then commit.**

   **2a — Write or update `plans/<CUR>.md`.** This file records what the branch
   changed and why. `/merge` refuses to land a branch without it, and this skill is
   what puts it there — so handle both cases:

   - **Missing** → **create it**, from the branch's full history. Do not assume
     `/change` ran: the branch may predate this convention, or have been cut by hand
     with `git checkout -b`. Create `plans/` if the repo has no such directory; a `/`
     in the branch name nests a subdirectory (`feature/x` → `plans/feature/x.md`).
   - **Present** → **update it.** `/change` seeds it with the branch's *intent*; this
     step makes it match what the branch actually became, so it stays accurate at
     every push rather than only at the end.

   Either way, do this **even when the working tree is clean.** A clean tree only
   means nothing is unsaved — commits made outside this skill still carry decisions
   the file may not record yet. Read what the branch has done:
   ```bash
   git log --oneline main..<CUR>
   git diff --stat main...<CUR>
   ```
   Then fold in everything since the file was last written: what changed and why,
   decisions taken and alternatives rejected, bugs found and their root causes, and
   how the work was verified. Prefer the reasoning over an inventory of files — the
   diff already records what changed and cannot record why. Where the branch
   outgrew the seeded intent, **correct the file** rather than appending to it; a
   plan that still describes an abandoned approach will be read as current.
   Include what the conversation established but the commits do not. Do not invent
   a rationale — if something's reasoning is genuinely unknown, say so.

   Write the plan for **`CUR` only**, never for other branches in the stack: this
   skill deliberately never checks out another branch, and writing theirs would
   require it. Each stacked branch gets its own plan the next time it is the
   checked-out branch being pushed; `/merge` validates the whole stack before
   landing it.

   **2b — Commit.** Inspect the working tree, which now includes any plan edit:
   ```bash
   git status --porcelain
   ```
   If it prints nothing, there is nothing to commit — skip to step 3. Otherwise
   there are uncommitted changes (staged, unstaged, and/or untracked files);
   stage and commit them all to the **current** branch before pushing:
   ```bash
   git add -A
   git commit -m "<message>"
   ```
   Write a concise message summarizing the change — inspect `git diff --cached
   --stat` and the changed files to describe it — and end it with any commit
   trailer this environment requires (e.g. the mandated `Co-Authored-By:` line).
   Do **not** switch branches. If a pre-commit hook rejects the commit, **stop
   and report**; never `--no-verify` around it.

   Note: `git add -A` stages everything in the working tree, including untracked
   files. If the user wants only some changes committed, they should stage or
   commit selectively themselves before invoking this skill.

3. **Detect the stack.** Determine whether the current branch (call it `CUR`) is
   part of a stack of local branches and, if so, the order from `main` upward.
   List local branch tips:
   ```bash
   git for-each-ref --format='%(refname:short) %(objectname)' refs/heads
   ```
   Relative to `CUR`:
   - **Branches below `CUR`** (branches it is stacked on): local branch `B` where
     `git merge-base --is-ancestor <B-tip> CUR` succeeds (and `B` != `CUR`, `B` !=
     `main`).
   - **Branches above `CUR`** (branches stacked on top of it): local branch `A`
     where `git merge-base --is-ancestor CUR <A-tip>` succeeds (and `A` != `CUR`).

   **Drop any below-branch that has already landed** in `main`:
   ```bash
   git merge-base --is-ancestor <B-tip> main    # succeeds => already landed
   ```
   Such a branch is a stale local ref, not part of the stack — its commits are
   already in `main` and it adds nothing over it. Pushing it **re-creates it on
   the remote** after a merge deleted it there, which is how a landed branch comes
   back from the dead. Skip it and mention it in the final report.

   (Fetch first if you want this to be accurate against a `main` that has moved:
   `git fetch origin main:main` fast-forwards `main` without checking it out. If
   local `main` is stale, a landed branch may not look landed yet — in which case
   it is pushed, which is harmless.)

   The stack is `main → (below branches) → CUR → (above branches)`. Order any set
   by ancestry: `X` comes before `Y` when
   `git merge-base --is-ancestor <X-tip> <Y-tip>` succeeds.

   If there are **no** below/above branches, it is a single branch — the stack is
   just `CUR`.

4. **Push each branch in the stack**, in ancestry order (bottom-most first, `CUR`,
   then any branches above it), setting upstream tracking automatically. For each
   branch `B` in the stack:
   ```bash
   git push -u origin <B>
   ```
   `git push -u origin <B>` works whether or not the branch already tracks a
   remote, and pushing the branch by name (rather than `HEAD`) means you do not
   have to check any of them out. **Never push `main`** — skip it if it ever
   appears here (stack detection already excludes it).

   **If any push is rejected** (e.g. the remote is ahead — a non-fast-forward),
   **stop immediately**: report which branch was rejected, do not push the
   remaining branches, and ask the user how they want to proceed (e.g.
   pull/rebase that branch first). Never add `--force` or `--force-with-lease` to
   get past a rejection.

5. Report the result: the plan-file update and the commit made in step 2 (if any),
   each branch pushed and that it succeeded, any PR-creation URL printed, and any
   below-branch skipped as already landed. If the process stopped on a rejection,
   report which branch was rejected and which branches (if any) were already
   pushed before it.

## Notes

- This skill **never pushes `main`**. If `main` is the current branch it stops at
  step 1 without committing or pushing, and `main` is skipped if it ever shows up
  in the stack. `main` should advance through merges/PRs, not a direct push.
- This skill updates `plans/<CUR>.md` and commits any pending changes to the
  current branch before pushing (step 2), staging everything with `git add -A`.
  Make sure the working tree holds only what you want committed — stage or commit
  selectively yourself first if not.
- Do not force-push. If a push is rejected because the remote is ahead, report the
  rejection and ask how the user wants to proceed — never add `--force` or
  `--force-with-lease` on your own.
- The stack-detection model here is the one the `rebase` and `merge` skills use,
  so all three agree on which branches make up the stack. The already-landed drop
  in step 3 is shared with `merge` (which must not open a PR for a branch with no
  commits of its own); `rebase` does not need it, since replaying a landed branch
  onto `main` is a no-op.
- Pushing by branch name (`git push -u origin <B>`) avoids any `checkout` dance —
  the working tree is never touched by the push and `CUR` stays checked out
  throughout.
