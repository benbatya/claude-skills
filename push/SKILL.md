---
name: push
description: Commit any uncommitted changes to the current branch, then push it to origin, setting upstream tracking if it is not yet configured. Never pushes the main branch. If the current branch is part of a stack of local branches, push every branch in the stack. Use when the user asks to push the current branch, "push this branch", "push my work", or "push the stack".
model: sonnet
---

# Push the current branch (stack-aware)

Push the checked-out git branch to the `origin` remote. Before pushing, commit
any uncommitted changes to the current branch so nothing in the working tree is
left behind. If the current branch is part of a **stack** (a chain of local
branches each built on the one below it), push every branch in the stack so all
their remotes advance together.

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

2. **Commit any uncommitted changes to the current branch.** Inspect the working
   tree:
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

5. Report the result: the commit made in step 2 (if any), each branch pushed and
   that it succeeded, and any PR-creation URL the remote prints for each. If the
   process stopped on a rejection, report which branch was rejected and which
   branches (if any) were already pushed before it.

## Notes

- This skill **never pushes `main`**. If `main` is the current branch it stops at
  step 1 without committing or pushing, and `main` is skipped if it ever shows up
  in the stack. `main` should advance through merges/PRs, not a direct push.
- This skill commits any pending changes to the current branch before pushing
  (step 2), staging everything with `git add -A`. Make sure the working tree
  holds only what you want committed — stage or commit selectively yourself first
  if not.
- Do not force-push. If a push is rejected because the remote is ahead, report the
  rejection and ask how the user wants to proceed — never add `--force` or
  `--force-with-lease` on your own.
- The stack-detection model here matches the `rebase` skill, so the two agree on
  which branches make up the stack.
- Pushing by branch name (`git push -u origin <B>`) avoids any `checkout` dance —
  the working tree is never touched by the push and `CUR` stays checked out
  throughout.
