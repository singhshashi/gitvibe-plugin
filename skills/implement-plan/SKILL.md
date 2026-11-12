---
name: implement-plan
description: Implement the draft plan for a local issue tracked in `git vibe`, given its issue number — takes `.gitvibe/plans/<N>-plan.md` as the accepted plan, implements it on a branch, and opens a git-vibe merge request. Use when asked to implement, execute, or carry out the plan for an issue. Does not draft plans; use the plan skill for that.
---

# implement-plan

Implement the draft plan for a tracked issue in the git-vibe local issue tracker, then open a merge request. Issues are inspected with the `git vibe` CLI (e.g. `git vibe show <n>`).

**The draft plan is the accepted plan.** This skill does not draft, discuss, or revise plans — the `plan` skill does that. Invoking this skill is the user's acceptance of the draft as it stands. At the end it opens the merge request on its own, without waiting to be asked.

## Issue number is required

The skill must be invoked with a git-vibe issue number (e.g. `/gitvibe:implement-plan 42`). If the request does not include one, tell the user the issue number is required and stop. Do not ask which issue to implement and do not pick one from `git vibe list`.

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

## Implementing issue #N

0. Run the **Preflight** above. If git-vibe is unavailable, stop.
1. Look for the draft plan at `.gitvibe/plans/N-plan.md` (exactly this path). If it does not exist, tell the user there is no draft plan for issue N, suggest creating one with the `plan` skill (`/gitvibe:plan N`), and stop. Do not write a plan yourself and do not look for one anywhere else.
2. Run `git vibe show N` to get the issue details, and read the draft plan in full.
3. Run `git vibe plan check .gitvibe/plans/N-plan.md` (see "Checking the plan"). If it does not exit 0, report the output, suggest fixing the draft with the `plan` skill, and stop. Do not repair the plan here — the user accepted the draft as written, not a version you changed.
4. Accept the plan: write the file's contents as a comment on the issue with `git vibe comment N --kind plan`, then delete `.gitvibe/plans/N-plan.md`. Do not alter the contents first. From here on `base_commit` is fixed: the review diff always compares it with the implementation, so never move it to hide unrelated changes.
5. Create an appropriately named git branch, and note the branch you created it from — the merge request in step 9 targets that branch, not the repo's trunk.
6. Implement the changes, one P-item at a time, keeping each edit inside the targets that P-item declared.
   - Do not force a correct fix into a range that turned out to be wrong. If the fix needs lines or files the plan did not declare, and the change is still within the agreed task, make it and note it for the summary. GitVibe will show it as **Not planned / Extra**, and that is expected.
   - If the change would expand the agreed scope in a material way (new behaviour, new components, unrelated files), stop and discuss it with the user before making it.
   - Never edit the accepted plan or its mapping block to make deviations disappear.
   - If you add temporary logging to debug the implementation, keep track of every line you add — step 7 removes them.
7. Once the changes are ready, remove all temporary logging you added — only those lines; leave the user's own uncommitted edits alone. Logging you deliberately keep must be part of a P-item or reported as extra in the summary. Then run the sanity checks (syntax, linting, tests) and the verification steps from the plan's section 7. If any fail and you cannot fix them within the plan's scope, stop and report — do not open the merge request.
8. Commit on the branch, then write an **implementation summary** on the issue with `git vibe comment N --kind summary` (see "The implementation summary").
9. Open the merge request without waiting to be asked: `git vibe merge open --source <branch> --target <base from step 5> --issue N`. Do not take the target from the session's "Main branch" hint — that names the repo's trunk, which is often not the branch the work forked from. Confirm with `git merge-base` if the base wasn't recorded. Then tell the user the merge request is open, list anything that still needs manual verification (for example GUI-only steps from the plan), and point out any **Not planned / Extra** work.

## Checking the plan

`git vibe plan check <plan-file>` validates the plan's `gitvibe-plan` block against the format rules and against the repository at `base_commit`. Run it from inside the repository being changed. It reads file contents from git objects and changes nothing. Act on its exit status:

- **0** — the plan is valid; continue. If it prints warnings (overlapping targets in different items), mention them to the user.
- **1** — the plan has errors. Report them and stop (step 3).
- **2** — the check could not finish (for example, `base_commit` is not in this repository). Report the message and stop; do not guess whether the plan is valid.

Never skip the check or validate the block by hand instead.

## The implementation summary

The summary is a separate comment (`git vibe comment N --kind summary`) written once the implementation is committed. It describes how the result differs from the accepted plan — deviations, additions, dropped steps, and why. If the implementation matches the plan exactly, say so explicitly.

Include a target reconciliation covering every target that was added, dropped, or changed:

| Target | Change | Reason | Actual |
|--------|--------|--------|--------|
| P2.1 | changed | the fix also needed the error branch | `src/cli/dispatch.rs` lines 118–140 |
| P2.4 | added | P2 needed a new test fixture | `tests/fixtures/plan.json` (add) |
| P1.3 | dropped | the legacy type is still used by the importer | — |

- **Target** — the accepted target's id, or a fresh id for a target the plan did not have (the next unused number in its item; never a dropped id).
- **Actual** — path and lines, or the file operation, in the **implementation commit**: state its full ID (`git rev-parse HEAD` after the final commit) above the table. Lines here refer to that commit, not to the base.

The summary is a record only. It does not change the plan's mapping, and work the plan did not declare stays **Not planned / Extra** in GitVibe.

Plans and summaries **cannot be edited later**, so get them right before writing.
