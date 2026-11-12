---
name: plan
description: Draft or refine a fix plan for a local issue tracked in `git vibe`, given its issue number, writing it to `.gitvibe/plans/<N>-plan.md` — planning only, never implementing. Use when asked to plan an issue, write or update a plan, or refine a plan without fixing it yet. Plans follow the fix-plan template of the fix-issue skill.
---

# plan

Write and refine the plan for a tracked issue in the git-vibe local issue tracker. Issues are inspected with the `git vibe` CLI (e.g. `git vibe show <n>`).

**This skill only plans.** Its sole output is the plan file `.gitvibe/plans/<N>-plan.md`. It never implements the plan: do not edit, create, or delete any other file, do not create branches, do not commit, and do not post comments to the issue. When the user is happy with the plan, they implement it with the `implement-plan` skill, which takes the plan file as the accepted plan (or with `fix-issue`, which picks it up as its draft).

## Preflight — is git-vibe available here?

This plugin can be installed globally, so it may run in a repo that has nothing to do with git-vibe. **Before any other step**, run `git vibe list`. Every failure below exits 1; match on the message:

- **`git: 'vibe' is not a git command. See 'git --help'.`** — the git-vibe CLI is not installed. Tell the user git-vibe is not available and stop.
- **``error: gitvibe not initialised — run `git vibe init` first``** — the CLI is installed but this repo has no git-vibe database. Tell the user, mention that `git vibe init` would set it up, and stop. Do **not** run `git vibe init` yourself.
- **`error: not inside a git repository`** — say so and stop.
- **Exit 0** — git-vibe is available; continue. (`No issues found.` is a valid exit-0 result, not an error.)

Then run `git vibe plan check --help`:

- **`error: unrecognized subcommand 'plan'`** (exit 2) — this git-vibe is too old to check plans. Tell the user to update git-vibe and stop.
- **Exit 0** — continue.

In every failure case, stop and report. Do not fall back to GitHub issues, a TODO file, or any other tracker, and do not guess at issue content.

## Issue number is required

The skill must be invoked with a git-vibe issue number (e.g. `/gitvibe:plan 42`). If the request does not include one, tell the user the issue number is required and stop. Do not ask which issue to plan, do not pick one from `git vibe list`, and do not plan without an issue.

## Template

The plan **MUST** follow `../fix-issue/templates/fix-plan.md` (the fix-issue skill's template, shared so both skills produce the same plan). Read it before drafting and match its sections exactly, omitting sections only where the template permits. It also defines the `gitvibe-plan` mapping block (GitVibe Plan Compatibility v1) and its rules.

## Planning issue #N

0. Run the **Preflight** above. If git-vibe is unavailable, stop.
1. If `.gitvibe/plans/N-plan.md` already exists, read it first — you are refining an earlier draft, not starting over (see "Refining an existing plan").
2. Run `git vibe show N` to get the issue details.
3. If the issue describes a bug or regression, confirm it exists before planning:
   - Read the relevant code path and check whether it plausibly produces the reported symptom. Note the specific lines/logic that explain it.
   - Check existing logs, and run the program or tests to reproduce the issue where that is possible without editing files.
   - Do **not** add temporary logging or any other edits to confirm a theory — this skill changes nothing but the plan file. If confirming the cause needs instrumentation, say so, and propose that step to the user instead.
   - If you find nothing that would cause the reported behavior, say so explicitly. Do not write a plan for an unconfirmed bug. Work with the user to reproduce it — ask for exact repro steps, the environment, and actual error output/logs.
4. Draft the plan into `.gitvibe/plans/N-plan.md` (create `.gitvibe/plans/` if needed):
   - Record the base commit **before** reading any code for coordinates: run `git rev-parse HEAD` in the repository being planned and put the full commit ID in the mapping block's `base_commit`. Never abbreviate it.
   - Take every path and line number from that commit, not from the working tree: read files with `git show <base_commit>:<path>` and count lines with `git show <base_commit>:<path> | awk 'END { print NR }'`.
   - The mapping block in section 4 is the only place paths, operations, and coordinates are written. Every P-item in the prose must be in the block with the same meaning, and the other way round.
   - Run `git vibe plan check` (see "Checking the plan") and fix every error before showing the plan to the user.
5. Discuss the plan with the user. Apply every change from the discussion to the plan file immediately — the file is the single current version of the plan. Run `git vibe plan check` after every revision.
   - Keep item ids and target ids stable while they describe the same work. Give new targets fresh ids and never reuse the id of a dropped target.
   - **The base stays put.** If `HEAD` moves while the plan is in draft, keep the original `base_commit`. Only change it if the user asks to rebase the plan; then recheck every target against the new base at once — never mix coordinates from two commits.
6. When the user is satisfied, run `git vibe plan check` one last time and stop. Tell the user the plan is at `.gitvibe/plans/N-plan.md` and that `implement-plan N` will implement it. Do not start implementing, even if the user's acceptance sounds like a go-ahead — if they ask to implement, tell them to invoke the `implement-plan` skill.

## The plan file

- **Path** — `.gitvibe/plans/<N>-plan.md`, where `<N>` is the git-vibe issue number. Use this name exactly.
- **Contents** — the full plan in the template format, and nothing else.
- **Purpose** — other software reads this file to show the plan to the user. Keep it current: do not keep a different version of the plan only in the conversation.
- **Lifetime** — this skill leaves the file in place. It is removed by `implement-plan` (or `fix-issue`) once that skill writes the accepted plan to the issue. Do not commit it, and do not post it with `git vibe comment`.

## Refining an existing plan

If the plan file already exists, it is an earlier draft for the same issue. Read it, then continue at step 5 with the user's requested changes, keeping its `base_commit` and ids. If you have not yet confirmed its targets in this session, re-verify them against its `base_commit` and run `git vibe plan check` before discussing it.

If that draft is in the older format (a `**Base revision:**` line and per-item change-site tables instead of a `gitvibe-plan` block), convert it to the current template first: keep its substance, turn each change-site row into a target (keeping its `P1.1`-style id; old `insert after N` / `before N` becomes a `point` with `after` N / N−1, an old line-range `delete` becomes `modify` with `lines`, and `add file` becomes `add`), expand the base revision to the full commit ID with `git rev-parse --verify '<short id>^{commit}'`, and re-verify every target against that commit. If the short ID does not resolve to exactly one commit, stop and ask the user.

## Checking the plan

`git vibe plan check <plan-file>` validates the plan's `gitvibe-plan` block against the format rules and against the repository at `base_commit`. Run it from inside the repository being planned, e.g. `git vibe plan check .gitvibe/plans/N-plan.md`. It reads file contents from git objects and changes nothing. Act on its exit status:

- **0** — the plan is valid. Warnings may still be printed: overlapping targets in different items make ownership of those lines ambiguous, so split the ranges where you can, and mention any you keep to the user.
- **1** — the plan has errors (`error: P2.3: …`). Fix them in the plan file and run the check again. Do not show the plan until it exits 0.
- **2** — the check could not finish (for example, `base_commit` is not in this repository). Report the message to the user and stop; do not guess whether the plan is valid.

Never skip the check or validate the block by hand instead.
