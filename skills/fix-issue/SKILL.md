---
name: fix-issue
description: Investigate, plan, and fix a local issue tracked in `git vibe`. Use when asked to fix a local issue or work through a git-vibe tracked issue end to end, including implementing a plan drafted with the plan skill. For planning only, without implementing, use the plan skill instead. Plans follow the fix-plan template in this skill.
---

# fix-issue

Fix a tracked issue in the git-vibe local issue tracker. Issues are tracked in a local database and inspected with the `git vibe` CLI (e.g. `git vibe show <n>`).

## Preflight — is git-vibe available here?

This plugin can be installed globally, so it may run in a repo that has nothing to do with git-vibe. **Before any other step**, run `git vibe list`. Every failure below exits 1; match on the message:

- **`git: 'vibe' is not a git command. See 'git --help'.`** — the git-vibe CLI is not installed. Tell the user git-vibe is not available and stop.
- **``error: gitvibe not initialised — run `git vibe init` first``** — the CLI is installed but this repo has no git-vibe database. Tell the user, mention that `git vibe init` would set it up, and stop. Do **not** run `git vibe init` yourself — it creates a `.gitvibe/` directory and prompts about `.gitignore`, so it is the user's call.
- **`error: not inside a git repository`** — say so and stop.
- **Exit 0** — git-vibe is available; continue. (`No issues found.` is a valid exit-0 result, not an error.)

Then run `git vibe plan check --help`. Plans are checked with this command (see "Checking the plan"), so the installed git-vibe must have it:

- **`error: unrecognized subcommand 'plan'`** (exit 2) — this git-vibe is too old to check plans. Tell the user to update git-vibe and stop.
- **Exit 0** — continue.

Then run `git vibe summary draft --help`. Implementation summaries are drafted from the implementation log (see "The implementation summary"), so the installed git-vibe must have `summary draft --log`:

- **`error: unrecognized subcommand 'summary'`** (exit 2), or **exit 0 with no `--log` option listed** — this git-vibe is too old. Tell the user to update git-vibe and stop.
- **Exit 0 with `--log` listed** — continue.

In every failure case, stop and report. Do not fall back to GitHub issues, a TODO file, or any other tracker, and do not guess at issue content.

## Templates

- `templates/fix-plan.md` — the required structure for any fix plan, including the `gitvibe-plan` mapping block (GitVibe Plan Compatibility v1) and its rules. Read it before drafting a plan and match its sections exactly.

## Fixing a local issue

To investigate and fix a tracked local issue #N:

0. Run the **Preflight** check above. If git-vibe is unavailable, stop there.
1. Run `git vibe show N` to get the issue details.
2. If the issue describes a bug or regression, confirm it exists before planning anything:
   - Read the relevant code path and check whether it plausibly produces the reported symptom. Note the specific lines/logic that explain the behavior.
   - Use logs generously: check what's already logged around that path, and if the existing logging doesn't confirm the theory, reproduce the issue and add temporary logging/tracing to observe actual behavior rather than guessing from code alone. Keep track of every temporary log line you add — step 8 removes them.
   - If you confirm a plausible root cause, proceed to step 3 with that root cause in hand — the plan's "Current State" section should cite it.
   - If, after this investigation, you find nothing in the code that would cause the reported behavior, say so explicitly. Do not write a plan for an unconfirmed bug. Instead, work with the user to reproduce it: ask for exact repro steps, the environment, and actual error output/logs, and use those to narrow down the cause together before planning a fix.
3. Investigate the relevant code and produce a plan. The plan **MUST** follow `templates/fix-plan.md` — read that file first and match its sections exactly, omitting sections only where the template permits. Write the plan to the **plan file** `.gitvibe/plans/N-plan.md` (see "The plan file" below) instead of only showing it in the conversation.
   - Record the base commit **before** reading any code for coordinates: run `git rev-parse HEAD` in the repository being fixed (not in this plugin's directory) and put the full commit ID in the mapping block's `base_commit`. Never abbreviate it.
   - Take every path and line number from that commit, not from the working tree: read files with `git show <base_commit>:<path>` and count lines with `git show <base_commit>:<path> | awk 'END { print NR }'`. Uncommitted edits — including any temporary logging from step 2 — shift working-tree line numbers, so the working tree is never a source of coordinates.
   - The mapping block in section 4 is the only place paths, operations, and coordinates are written. Do not add tables or lists that repeat them. Every P-item in the prose must be in the block with the same meaning, and the other way round.
   - Run `git vibe plan check` on the plan file (see "Checking the plan") and fix every error before showing the plan to the user.
4. Discuss the plan with the user. Make every change that comes out of the discussion in the plan file — keep the file the single current version of the plan. Run `git vibe plan check` again after every revision.
   - Keep item ids and target ids stable while they describe the same work. Give new targets fresh ids and never reuse the id of a dropped target.
   - **The base stays put.** If `HEAD` moves while the plan is in draft (for example the user commits), keep the original `base_commit`. Only change it if the user asks to rebase the plan; then recheck every target against the new base at once — never mix coordinates from two commits — and show the revised plan to the user before it can be accepted.
5. Once the user accepts the plan, run `git vibe plan check` one last time, then write the file's contents as a comment on the issue with `git vibe comment N --kind plan`, then delete `.gitvibe/plans/N-plan.md`. From here on `base_commit` is fixed: the review diff always compares it with the implementation, so never move it to hide unrelated changes.
6. Create an appropriately named git branch, and note the branch you created it from — the merge request in step 9 targets that branch, not the repo's trunk.
7. Implement the changes, verify the implementation where possible, then ask for manual review and verification. Work one P-item at a time and keep each edit inside the targets that P-item declared.
   - Keep the implementation log as you go: one entry when you start each P-item, and one when new work comes up (see "The implementation log").
   - Do not force a correct fix into a range that turned out to be wrong. If the fix needs lines or files the plan did not declare, and the change is still within the agreed task, log it as new work with its `why` and make it. The summary will own it as an S-item.
   - If the change would expand the agreed scope in a material way (new behaviour, new components, unrelated files), stop and discuss it with the user before making it.
   - Never edit the accepted plan or its mapping block to make differences disappear.
8. Once the changes are ready for review, first remove all temporary logging from step 2 — only the lines you added; leave the user's own uncommitted edits alone. Logging you deliberately keep must be part of a P-item or be logged as new work, with its `why`, so the summary can own it. Then run the sanity checks (syntax, linting, tests) and commit on the branch. Once the implementation is complete, draft, check and post an **implementation summary** with `git vibe summary draft --log` → `git vibe summary check` → `git vibe comment N --kind summary` (see "The implementation summary"). After any later commits on the branch (review fixes, for example), redraft and post a full snapshot the same way.
9. If asked to open a merge request, run `git vibe merge open --source <branch> --target <base from step 6> --issue N`. Do not take the target from the session's "Main branch" hint — that names the repo's trunk, which is often not the branch the work forked from. Confirm with `git merge-base` if the base wasn't recorded.

## The plan file

While a plan is in draft, it lives in a file, not in the conversation:

- **Path** — `.gitvibe/plans/<N>-plan.md`, where `<N>` is the git-vibe issue number (e.g. issue 42 → `.gitvibe/plans/42-plan.md`). Use this name exactly. Create the `.gitvibe/plans/` directory if it does not exist.
- **Contents** — the full plan in the `templates/fix-plan.md` format, and nothing else.
- **Purpose** — other software reads this file to show the plan to the user. Keep it current at all times: apply each revision from the discussion to the file immediately, and do not keep a different version of the plan only in the conversation.
- **Lifetime** — the file is temporary. It exists from the first draft until the user accepts the plan. After you write the accepted plan to the database with `git vibe comment N --kind plan`, delete the file. Do not commit it.

If a plan file for the issue already exists when you start, read it first — it is an earlier draft for the same issue. If that draft is in the older format (a `**Base revision:**` line and per-item change-site tables instead of a `gitvibe-plan` block), convert it to the current template before continuing: keep its substance, turn each change-site row into a target (keeping its `P1.1`-style id; old `insert after N` / `before N` becomes a `point` with `after` N / N−1, an old line-range `delete` becomes `modify` with `lines`, and `add file` becomes `add`), expand the base revision to the full commit ID with `git rev-parse --verify '<short id>^{commit}'`, and re-verify every target against that commit. If the short ID does not resolve to exactly one commit, stop and ask the user. Plans already written to the issue with `--kind plan` are never converted.

## Checking the plan

`git vibe plan check <plan-file>` validates the plan's `gitvibe-plan` block against the format rules and against the repository at `base_commit`. Run it from inside the repository being fixed, e.g. `git vibe plan check .gitvibe/plans/N-plan.md`. It reads file contents from git objects, so uncommitted edits don't affect it, and it changes nothing.

Run it after writing the first draft (step 3), after every revision (step 4), and just before posting the accepted plan (step 5). Act on its exit status:

- **0** — the plan is valid. Warnings may still be printed: overlapping targets in different items make ownership of those lines ambiguous, so split the ranges where you can, and mention any you keep to the user.
- **1** — the plan has errors. Each line names the item or target (`error: P2.3: …`). Fix them in the plan file and run the check again. Do not show or post the plan until it exits 0.
- **2** — the check could not finish (for example, `base_commit` is not in this repository, or a file could not be read). Report the message to the user and stop; do not guess whether the plan is valid.

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

Comments take a `--kind` (default `comment`): use `plan` for the accepted plan, `summary` for the implementation summary, and the default `comment` for anything else. Plans and summaries **cannot be edited later**, so get them right before writing — write them once the plan is accepted / the implementation is complete, not before.
