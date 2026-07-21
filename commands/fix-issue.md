---
argument-hint: [issue-number]
description: Fix a local issue found in git vibe
---
Fix local issue #$1 in this repo.

Use the **gitvibe** skill. Run its **Preflight** check first — if the `git vibe` CLI is missing or this repo has no git-vibe database, say so and stop without doing anything else.

If it passes, follow the skill's "Fixing a local issue" workflow end to end — investigate with `git vibe show $1`, produce a plan that matches the skill's `fix-plan` template exactly, write the accepted plan as a comment on the issue, branch, implement, and verify.
