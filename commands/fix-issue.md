---
argument-hint: [issue-number]
description: Fix a local issue found in git vibe
---
Fix local issue #$1 in this repo.

Use the **gitvibe** skill. Run its **Preflight** check first — if the `git vibe` CLI is missing or this repo has no git-vibe database, say so and stop without doing anything else.

If it passes, follow the skill's "Fixing a local issue" workflow end to end — investigate with `git vibe show $1`, produce a plan that matches the skill's `fix-plan` template exactly, write the draft plan to `.gitvibe/plans/$1-plan.md` and keep that file current during the discussion, write the accepted plan as a comment on the issue and delete the file, branch, implement, and verify. When implementation is complete, write an implementation summary comment on the issue focusing on how the final implementation differs from the accepted plan.
