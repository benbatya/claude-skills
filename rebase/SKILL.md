---
name: rebase
description: Fast-forward local main to the latest remote, then rebase the current branch (and any branches stacked with it) on top of it, in stacked order. Use when the user asks to "update main and rebase", "sync main and rebase my branch", or "rebase onto latest main".
---

# Fast-forward main, then rebase the current branch (stack-aware)

Bring local `main` up to date with the remote, then replay the checked-out
branch on top of it. If the current branch is part of a **stack** (a chain of
local branches each built on the one below it), rebase every branch in the
stack in order so the stack stays intact.

## Steps

1. Capture the starting branch:
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```
   If this returns `HEAD`, the repo is in detached-HEAD state — stop and tell the
   user there is no branch to rebase. If it is already `main`, stop and say so.

2. Guard against a dirty working tree. Check only tracked, uncommitted changes —
   untracked files do not interfere with a fetch, checkout, or rebase, so they
   must not block the skill:
   ```bash
   git status --porcelain --untracked-files=no
   ```
   If this prints anything, there are uncommitted tracked changes (modified or
   staged). **Stop and report** — ask the user to commit or stash first. Do
   **not** auto-stash, discard, or `git checkout -- .` their work.

3. Pick the upstream remote and fetch it. Prefer an `upstream` remote if one
   exists, otherwise use `origin`:
   ```bash
   git remote                 # use "upstream" if listed, else "origin"
   git fetch <remote> --prune
   ```

4. Fast-forward local `main` without leaving the current branch, using a fetch
   refspec (this works even while `main` is not checked out and never touches the
   working tree):
   ```bash
   git fetch <remote> main:main
   ```
   If this fails with a non-fast-forward error, local `main` has diverged from the
   remote — **stop and report**. Never force it.

5. **Detect the stack.** Determine whether the current branch is part of a stack
   of local branches and, if so, the order from `main` upward. List local branch
   tips:
   ```bash
   git for-each-ref --format='%(refname:short) %(objectname)' refs/heads
   ```
   Relative to the current branch (call it `CUR`):
   - **Branches below `CUR`** (branches it is stacked on): local branch `B` where
     `git merge-base --is-ancestor <B-tip> CUR` succeeds (and `B` != `CUR`, `B` !=
     `main`).
   - **Branches above `CUR`** (branches stacked on top of it): local branch `A`
     where `git merge-base --is-ancestor CUR <A-tip>` succeeds.

   The stack is `main → (below branches, ancestor→descendant) → CUR → (above
   branches, ancestor→descendant)`. Order any set by ancestry: `X` comes before
   `Y` when `git merge-base --is-ancestor <X-tip> <Y-tip>` succeeds.

   If there are **no** below/above branches, it is a single branch — skip to the
   plain rebase in step 6b.

6. **Rebase the stack in order.**

   **6a — Linear stack (preferred, uses `git rebase --update-refs`).** Most
   stacks are linear: each branch's tip is an ancestor of the next, so the
   topmost branch's history contains every lower branch tip. In that case one
   rebase of the **top** branch updates the whole chain:
   ```bash
   git checkout <top-of-stack>       # the branch that has all others as ancestors
   git rebase --update-refs main     # rebases the chain, moves every intermediate branch ref
   ```
   `--update-refs` (git ≥ 2.38; this repo's git supports it) automatically
   fast-forwards each intermediate stack branch ref to its new commit, so every
   branch below the top lands in the right place in one operation.

   Confirm the stack is linear before using 6a: the top branch is the one for
   which every other stack branch tip satisfies
   `git merge-base --is-ancestor <other-tip> <top-tip>`. If no single branch
   dominates all others (the stack forks into multiple leaves — a tree), use 6b.

   **6b — Single branch, or a forked/tree-shaped stack (explicit per-branch).**
   Rebase each branch onto its already-rebased parent, bottom-up. For each branch
   `B` with parent `P` (its immediate lower neighbour; `main` for the lowest),
   **record `B`'s old tip first**, then move only `B`'s own commits with the
   three-arg `--onto` form so already-rebased parent commits are not duplicated:
   ```bash
   OLD_TIP=$(git rev-parse <B>)            # capture BEFORE moving P
   # after P has been rebased:
   git rebase --onto <P> $OLD_TIP <B>      # replay B's commits onto the new P
   ```
   Process branches strictly in ancestor→descendant order (parents before
   children) so each `--onto <P>` targets an already-updated parent. A single
   branch with nothing stacked on it is just `git checkout CUR && git rebase main`.

   **Conflicts (either path):** **stop**, report the branch being rebased and the
   conflicted files, and tell the user to resolve them and run
   `git rebase --continue`, or `git rebase --abort` to back out. Never resolve
   conflicts, `git rebase --skip`, or `git rebase --abort` on the user's behalf.

7. **Return to the starting branch** captured in step 1:
   ```bash
   git checkout <starting-branch>
   ```

8. Report the result: which remote was used, `main`'s new position (short SHA and
   roughly how far it advanced), and — for each branch in the stack, in order —
   its new tip, confirming the stack was rebased intact. If the process stopped on
   a conflict, report which branch it stopped on and the exact next step.

## Notes

- Never force-push and never run `git push --force` / `--force-with-lease`.
- Never auto-stash, discard, or `git checkout -- .` the user's uncommitted work.
- Never `git rebase --skip` / `--abort` or resolve conflicts autonomously — hand
  control back to the user.
- This skill does **not** push. Rebasing rewrites history for the current branch
  **and every branch stacked above it**, so updating their remotes would require a
  force-push, which this toolchain deliberately will not do automatically. If the
  user wants the stack pushed, that is a separate, explicit action.
- Step 4 uses the `main:main` fetch refspec on purpose: it fast-forwards `main`
  without a `checkout main` / `checkout back` dance and without disturbing the
  working tree.
- Prefer `--update-refs` (6a) for linear stacks — it is atomic per rebase and
  keeps every intermediate ref consistent. Only fall back to the per-branch
  `--onto` walk (6b) when the stack forks into multiple leaves.
