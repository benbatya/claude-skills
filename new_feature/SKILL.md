---
name: new_feature
description: Alias for the `change` skill. Start a fresh change from the latest main. Switch to main, fast-forward it to the upstream remote, create a new branch named YYMMDD_<description> derived from the request, then begin the work (propose a plan or author code) from the prompt that followed the invocation. Use when the user says "/new_feature ...", "start a new feature", "begin new work", or wants to kick off a task on a clean branch off main.
---

# Start new work on a fresh branch off the latest main (alias of `change`)

`/new_feature` is an alias for the **`change`** skill. There is exactly one set of
instructions, and it lives in that skill — this file deliberately does not repeat it.

**Do this now:**

1. Read `/home/benbatya/.claude/skills/change/SKILL.md`.
2. Follow it start to finish, exactly as written.
3. Wherever it refers to the text following `/change` (the **change prompt**), use
   the text following `/new_feature` instead — that text both names the branch and
   drives the work in the final step.

Do not paraphrase or re-derive the steps here; `change/SKILL.md` is the single
source of truth. If it changes, this alias picks the change up automatically.
