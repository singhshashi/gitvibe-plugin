---
name: gitvibe
description: Work with git-vibe local issue tracking — investigate, plan, and fix local issues tracked in `git vibe`. Use when asked to fix a local issue, plan a fix, or work through a git-vibe tracked issue. Plans follow the fix-plan template in this skill.
---

# gitvibe

Workflows for the git-vibe local issue tracker. Issues are tracked in a local database and inspected with the `git vibe` CLI (e.g. `git vibe show <n>`).

## Preflight — is git-vibe available here?

This plugin can be installed globally, so it may run in a repo that has nothing to do with git-vibe. **Before any other step**, run `git vibe list`. Every failure below exits 1; match on the message:

- **`git: 'vibe' is not a git command. See 'git --help'.`** — the git-vibe CLI is not installed. Tell the user git-vibe is not available and stop.
- **``error: gitvibe not initialised — run `git vibe init` first``** — the CLI is installed but this repo has no git-vibe database. Tell the user, mention that `git vibe init` would set it up, and stop. Do **not** run `git vibe init` yourself — it creates a `.gitvibe/` directory and prompts about `.gitignore`, so it is the user's call.
- **`error: not inside a git repository`** — say so and stop.
- **Exit 0** — git-vibe is available; continue. (`No issues found.` is a valid exit-0 result, not an error.)

In every failure case, stop and report. Do not fall back to GitHub issues, a TODO file, or any other tracker, and do not guess at issue content.

## Templates

- `templates/fix-plan.md` — the required structure for any fix plan. Read it before drafting a plan and match its sections exactly.

## Fixing a local issue

To investigate and fix a tracked local issue #N:

0. Run the **Preflight** check above. If git-vibe is unavailable, stop there.
1. Run `git vibe show N` to get the issue details.
2. Investigate the relevant code and produce a plan. The plan **MUST** follow `templates/fix-plan.md` — read that file first and match its sections exactly, omitting sections only where the template permits.
3. Once the plan is accepted, write it (in the template's format) as a comment on the issue.
4. Create an appropriately named git branch.
5. Implement the changes, verify the implementation where possible, then ask for manual review and verification.

> The `/gitvibe:fix-issue` slash command is a thin wrapper around this workflow.
