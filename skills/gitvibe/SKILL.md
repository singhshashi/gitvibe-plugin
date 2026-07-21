---
name: gitvibe
description: Work with git-vibe local issue tracking — investigate, plan, and fix local issues tracked in `git vibe`. Use when asked to fix a local issue, plan a fix, or work through a git-vibe tracked issue. Plans follow the fix-plan template in this skill.
---

# gitvibe

Workflows for the git-vibe local issue tracker. Issues are tracked in a local database and inspected with the `git vibe` CLI (e.g. `git vibe show <n>`).

## Templates

- `templates/fix-plan.md` — the required structure for any fix plan. Read it before drafting a plan and match its sections exactly.

## Fixing a local issue

To investigate and fix a tracked local issue #N:

1. Run `git vibe show N` to get the issue details.
2. Investigate the relevant code and produce a plan. The plan **MUST** follow `templates/fix-plan.md` — read that file first and match its sections exactly, omitting sections only where the template permits.
3. Once the plan is accepted, write it (in the template's format) as a comment on the issue.
4. Create an appropriately named git branch.
5. Implement the changes, verify the implementation where possible, then ask for manual review and verification.

> The `/gitvibe:fix-issue` slash command is a thin wrapper around this workflow.
