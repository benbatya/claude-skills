---
name: change
description: Start a fresh change from the latest main. Switch to main, fast-forward it to the upstream remote, create a new branch named YYMMDD_<description> derived from the request, then begin the work (propose a plan or author code) from the prompt that followed the invocation. Use when the user says "/change ...", "start a new change", "begin new work", or wants to kick off a task on a clean branch off main.
---

# Start a new change on a fresh branch off the latest main

Prepare a clean starting point for a new piece of work: get onto an up-to-date
`main`, cut a dated feature branch for it, then start the actual task described
in the prompt.

The text following the `/change` invocation (the **change prompt**) describes
what to build. Use it both to **name the branch** and to **drive the work** in
the final step.

## Steps

1. Determine the current branch:
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```
   If it returns `HEAD`, the repo is in a detached-HEAD state — stop and tell the
   user; do not cut a branch from a detached HEAD.

2. Guard against a dirty working tree. Check only tracked, uncommitted changes
   (untracked files do not interfere with a fetch or checkout):
   ```bash
   git status --porcelain --untracked-files=no
   ```
   If this prints anything, there are uncommitted tracked changes. **Stop and
   report** — ask the user to commit or stash first. Do **not** auto-stash,
   discard, `git checkout -- .`, or silently carry the changes onto the new
   branch.

3. Pick the upstream remote and fetch it. Prefer an `upstream` remote if one
   exists, otherwise `origin`:
   ```bash
   git remote                 # use "upstream" if listed, else "origin"
   git fetch <remote> --prune
   ```
   If the repo has no remote at all, skip the fetch and fast-forward (steps 3–5),
   note that `main` can't be updated (no remote), and continue from the local
   `main`.

4. Switch to `main` if not already there (the tree is clean per step 2):
   ```bash
   git checkout main
   ```

5. Fast-forward `main` to the upstream remote — never a merge commit, never a
   force:
   ```bash
   git pull --ff-only <remote> main
   ```
   If this fails with a non-fast-forward error, local `main` has diverged from
   the remote — **stop and report**. Do not reset, rebase, or force it.

6. Create and switch to the new branch, named `YYMMDD_<description>`:
   - **Date prefix** — today as two-digit year/month/day:
     ```bash
     date +%y%m%d
     ```
   - **Description** — a concise, lowercase `snake_case` slug summarizing the
     change prompt: strip punctuation, join ~2–5 keywords with underscores (e.g.
     "make the default speed x480" → `default_speed_480`). Keep it short and
     descriptive.

   If the change prompt is empty (bare `/change` with no description), ask
   the user for a one-line description of the intended change before naming the
   branch — do not guess a name.

   Then:
   ```bash
   git checkout -b <YYMMDD>_<description>
   ```
   If a branch with that exact name already exists, adjust the description (add a
   distinguishing word or a numeric suffix) so the name is unique — never
   overwrite or `-f` an existing branch.

7. Confirm the setup briefly: the remote used, `main`'s new position, and the
   new branch name.

8. **Begin the work.** Now act on the change prompt that followed `/change`:
   - For a non-trivial or ambiguous change, propose a plan first and check it
     with the user before writing code.
   - For a small, well-specified change, author the code directly.

   Follow the user's stated preferences about planning vs. implementing, and this
   repo's conventions (e.g. its build/test workflow) once you start coding.

## Notes

- Never force-push, `git reset --hard`, `git checkout -- .`, auto-stash, or
  discard uncommitted work.
- Never force the fast-forward: if `main` has diverged, stop and report.
- This skill does **not** commit, push, or open a PR — it only prepares the
  branch and starts the work. Committing/pushing are separate, explicit actions.
- Branch names use underscores throughout (`YYMMDD_snake_case`), matching the
  requested format.
