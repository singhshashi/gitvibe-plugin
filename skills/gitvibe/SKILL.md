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
2. If the issue describes a bug or regression, confirm it exists before planning anything:
   - Read the relevant code path and check whether it plausibly produces the reported symptom. Note the specific lines/logic that explain the behavior.
   - Use logs generously: check what's already logged around that path, and if the existing logging doesn't confirm the theory, reproduce the issue and add temporary logging/tracing to observe actual behavior rather than guessing from code alone.
   - If you confirm a plausible root cause, proceed to step 3 with that root cause in hand — the plan's "Current State" section should cite it.
   - If, after this investigation, you find nothing in the code that would cause the reported behavior, say so explicitly. Do not write a plan for an unconfirmed bug. Instead, work with the user to reproduce it: ask for exact repro steps, the environment, and actual error output/logs, and use those to narrow down the cause together before planning a fix.
3. Investigate the relevant code and produce a plan. The plan **MUST** follow `templates/fix-plan.md` — read that file first and match its sections exactly, omitting sections only where the template permits.
4. Once the plan is accepted, write it (in the template's format) as a comment on the issue with `git vibe comment N --kind plan`.
5. Create an appropriately named git branch, and note the branch you created it from — the merge request in step 8 targets that branch, not the repo's trunk.
6. Implement the changes, verify the implementation where possible, then ask for manual review and verification.
7. Once the changes are ready for review — sanity checks such as syntax, linting, and tests pass — commit them on the branch. Once the implementation is complete, write an **implementation summary** on the issue with `git vibe comment N --kind summary`. Focus it on how the final implementation differs from the accepted plan — deviations, additions, dropped steps, and why. If it matched the plan exactly, say so.
8. If asked to open a merge request, run `git vibe merge open --source <branch> --target <base from step 5> --issue N`. Do not take the target from the session's "Main branch" hint — that names the repo's trunk, which is often not the branch the work forked from. Confirm with `git merge-base` if the base wasn't recorded.

Comments take a `--kind` (default `comment`): use `plan` for the accepted plan, `summary` for the implementation summary, and the default `comment` for anything else. Plans and summaries **cannot be edited later**, so get them right before writing — write them once the plan is accepted / the implementation is complete, not before.

> The `/gitvibe:fix-issue` slash command is a thin wrapper around this workflow.
