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

Then run `git vibe summary draft --help`. Implementation summaries are drafted from the implementation log (see "The implementation summary"), so the installed git-vibe must have `summary draft --log`:

- **`error: unrecognized subcommand 'summary'`** (exit 2), or **exit 0 with no `--log` option listed** — this git-vibe is too old. Tell the user to update git-vibe and stop.
- **Exit 0 with `--log` listed** — continue.

In every failure case, stop and report. Do not fall back to GitHub issues, a TODO file, or any other tracker, and do not guess at issue content.

## Implementing issue #N

0. Run the **Preflight** above. If git-vibe is unavailable, stop.
1. Look for the draft plan at `.gitvibe/plans/N-plan.md` (exactly this path). If it does not exist, tell the user there is no draft plan for issue N, suggest creating one with the `plan` skill (`/gitvibe:plan N`), and stop. Do not write a plan yourself and do not look for one anywhere else.
2. Run `git vibe show N` to get the issue details, and read the draft plan in full.
3. Run `git vibe plan check .gitvibe/plans/N-plan.md` (see "Checking the plan"). If it does not exit 0, report the output, suggest fixing the draft with the `plan` skill, and stop. Do not repair the plan here — the user accepted the draft as written, not a version you changed.
4. Accept the plan: write the file's contents as a comment on the issue with `git vibe comment N --kind plan`, then delete `.gitvibe/plans/N-plan.md`. Do not alter the contents first. From here on `base_commit` is fixed: the review diff always compares it with the implementation, so never move it to hide unrelated changes.
5. Create an appropriately named git branch, and note the branch you created it from — the merge request in step 9 targets that branch, not the repo's trunk.
6. Implement the changes, one P-item at a time, keeping each edit inside the targets that P-item declared.
   - Keep the implementation log as you go: one entry when you start each P-item, and one when new work comes up (see "The implementation log").
   - Do not force a correct fix into a range that turned out to be wrong. If the fix needs lines or files the plan did not declare, and the change is still within the agreed task, log it as new work with its `why` and make it. The summary will own it as an S-item.
   - If the change would expand the agreed scope in a material way (new behaviour, new components, unrelated files), stop and discuss it with the user before making it.
   - Never edit the accepted plan or its mapping block to make differences disappear.
   - If you add temporary logging to debug the implementation, keep track of every line you add — step 7 removes them.
7. Once the changes are ready, remove all temporary logging you added — only those lines; leave the user's own uncommitted edits alone. Logging you deliberately keep must be part of a P-item or be logged as new work, with its `why`, so the summary can own it. Then run the sanity checks (syntax, linting, tests) and the verification steps from the plan's section 7. If any fail and you cannot fix them within the plan's scope, stop and report — do not open the merge request.
8. Commit on the branch, then draft, check and post an **implementation summary** with `git vibe summary draft --log` → `git vibe summary check` → `git vibe comment N --kind summary` (see "The implementation summary"). After any later commits on the branch (review fixes, for example), redraft and post a full snapshot the same way.
9. Open the merge request without waiting to be asked: `git vibe merge open --source <branch> --target <base from step 5> --issue N`. Do not take the target from the session's "Main branch" hint — that names the repo's trunk, which is often not the branch the work forked from. Confirm with `git merge-base` if the base wasn't recorded. Then tell the user the merge request is open, list anything that still needs manual verification (for example GUI-only steps from the plan), and point out the work found during implementation (the S-items) and any dropped or relocated targets.

## Checking the plan

`git vibe plan check <plan-file>` validates the plan's `gitvibe-plan` block against the format rules and against the repository at `base_commit`. Run it from inside the repository being changed. It reads file contents from git objects and changes nothing. Act on its exit status:

- **0** — the plan is valid; continue. If it prints warnings (overlapping targets in different items), mention them to the user.
- **1** — the plan has errors. Report them and stop (step 3).
- **2** — the check could not finish (for example, `base_commit` is not in this repository). Report the message and stop; do not guess whether the plan is valid.

Never skip the check or validate the block by hand instead.

## The implementation log

While implementing, record what each change is *for* in `.gitvibe/summaries/N-log.jsonl` (create the directory if needed). You know whether an edit is planned or new work when you make it; reconstructing that from the diff later, after context compaction or in another session, is guesswork. The log is your working file: never post it, never commit it. It holds intent only, so put no line numbers in it.

Write **at boundaries, not after every edit**: one entry when you start each P-item (or target), and one when work comes up that the plan did not anticipate. Five edits for P2 are one entry. Keep each entry to about 50 tokens. Append one JSON object per line; never overwrite the file.

| Field | Meaning |
|---|---|
| `item` | Required. The accepted P-item or target id you are starting (`P2`, `P2.2`), or `"new"` for work the plan did not anticipate. |
| `why` | Required with `"new"`: why the work was needed. It becomes the S-item's description. |
| `for` | Optional, with `"new"` only: the P-item the work is needed for. Omit it for work tied to no P-item, such as formatter churn. |
| `paths` | Optional: repository-relative files the work touches. |

```json
{"item": "P2", "paths": ["src/review.rs"]}
{"item": "new", "for": "P3", "paths": ["src/review.rs"], "why": "Every Owned literal must set the new explained field"}
{"item": "new", "why": "cargo fmt rewrapped lines in files this change touched"}
```

The log lives as long as the merge request, not the session. Later sessions, such as review fixes, append to the same file. Do not delete it after posting a summary; delete it once the merge request is merged or closed.

## The implementation summary

The summary is a separate comment (`git vibe comment N --kind summary`), written once the implementation is committed. Its `gitvibe-summary` block gives every change the plan does not own an owner that says why it exists, and says what became of accepted targets that were not implemented as declared. GitVibe reads only the block. Never work out its coordinates by hand: they are base-side and come from `git vibe summary draft`.

1. **Draft.** After the final commit, run `git vibe summary draft N --log .gitvibe/summaries/N-log.jsonl > .gitvibe/summaries/N-summary.md` (the draft is for `HEAD`). stdout is exactly the document to post. stderr has the notes: what was carried forward or left out, relocation hints, the log entries beside the open changes, and warnings. Read them all. If it exits 2, stderr says why:
   - **The log has an invalid line** — nothing was drafted. Correct the line it names and draft again. Never drop the entry.
   - **The log does not exist** — nothing was drafted. Do not make one up after the fact. Tell the user, and ask whether to draft without `--log`.
   - **Evidence is missing** (for example, the plan base is not in this repository) — it printed a prose template with no block. Tell the user the reason and continue with the prose.
   - **Anything else** — report it and stop.
2. **Assign every open change.** The draft carries the earlier summary forward, and adds an S-item with an empty `description` for each open change run and a `dropped` entry with an empty `reason` for each target with no matching change. Decide each from the log, with this test: *would a reviewer reading the P-item's description expect this edit?*
   - **Yes: the planned edit landed outside its target's declared coordinates, in the same file with the same operation.** Turn the S-item into a relocation: move its locations, without `id`, into the `at` of a `relocated` entry for that target in `plan_targets`, and remove the S-item. This covers a block git placed a gap away from the declared point, a planned edit that spilled onto an adjacent line, and planned work written elsewhere in the file. Replace the target's `dropped` entry if it has one, and add to its `relocated` entry if it already has one: a target id appears only once. `reason` is optional.
   - **No: it is separate work the plan did not foresee.** Keep the S-item. Use the logged `why` as its `description`, and set `for` when the work was needed for a P-item; omit it otherwise. Call sites and tests updated because of a planned change, a new import, and new fixtures or files are S-items. So is any work in another file, or of another operation than the target's.
   - **The log does not explain it.** Nobody recorded intending it, so look at the change again before giving it a reason. If you did not intend it, remove it, commit, and draft again.
   - **A target really was not implemented.** Keep its `dropped` entry and fill in the `reason`.

   ```json
   {"id": "P2.2", "status": "relocated",
    "at": [{"path": "src/review.rs", "operation": "modify", "scope": {"kind": "point", "after": 42}}],
    "reason": "Git placed the inserted block one gap after the declared point"}
   ```

   Then:
   - Merge S-items that share a reason: move their targets into one item and renumber those targets under it (`S1.1`, `S1.2`). Also merge or fix whatever the notes say must be resolved first. A carried-forward S-item keeps its id.
   - Claim the whole of each run. Never invent targets for changes the draft did not list, and never change `plan_base_commit`, `implementation_commit` or any coordinates.
   - If the plan has no identifiable items, the draft has no `for` and no `plan_targets` entries. Do not add any.
   - Replace the prose placeholder with a readable summary. Explain in prose any change the notes say no target can claim. If `items` and `plan_targets` are both empty, say explicitly that the implementation matches the plan. A table, if you include one, is derived from the block.
3. **Check.** Run `git vibe summary check .gitvibe/summaries/N-summary.md --issue N`, and act on its exit status:
   - **0** — continue.
   - **1** — the summary has mistakes its author can fix, including any change it leaves unexplained, any drop it does not disclose and any target that claims no change. Fix the summary and check again. Never post a summary that exits 1.
   - **2** — the check could not complete. Continue, but tell the user why, and do not claim the summary was verified.
4. **Post.** Run `git vibe comment N --kind summary -F .gitvibe/summaries/N-summary.md`, then delete that file (keep the log). Post also on exit 2 and when there are warnings, including when the plan is not compatible.
5. **Report.** Report every remaining warning from the check to the user, with the reason you left each one.

Do not change the implementation or the accepted plan to silence a diagnostic. The one exception is removing a change you did not intend; commit that, then draft again.

**After later commits** (review fixes, for example), log the new work as you do it, then repeat all the steps. Each summary is a full snapshot and only one is ever applied, so post the whole redrafted block, including everything the draft carried forward.

Plans and summaries **cannot be edited later**, so get them right before writing.
