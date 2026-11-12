# gitvibe — Claude Code & OpenAI Codex plugin

A plugin for the **git-vibe** local issue tracker. It bundles a set of skills, each named for its task; **`fix-issue`** — an investigate → plan → fix workflow with a required `fix-plan` template — and **`plan`**, which only drafts and refines that plan in `.gitvibe/plans/<N>-plan.md` and never implements it. A plan written with `plan` is picked up as the draft when you later run `fix-issue` on the same issue. Each skill carries its entire workflow, so every agent invokes the skill directly; there is no separate command wrapper.

This repository is both the plugin and its marketplace. **Both agents use the exact same skill implementation** — e.g. `skills/fix-issue/SKILL.md` and `skills/fix-issue/templates/fix-plan.md`. There is no duplicated copy for Codex: there is **one** marketplace manifest (`.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`), and both Claude Code and Codex install from it. Harness-specific metadata lives next to each skill in `skills/<skill>/agents/` so it travels with the skill, which is how more agents will be added.

## Claude Code

### Install

```
/plugin marketplace add singhshashi/gitvibe-plugin
/plugin install gitvibe@gitvibe
```

`gitvibe@gitvibe` = the plugin named `gitvibe` from the marketplace named `gitvibe`.

### Usage

Invoke the skill either way:

- **By name:** `/gitvibe:fix-issue 47` — `<plugin>:<skill>`; the text after it (`47`) is your request.
- **By natural language:** `fix local issue 47` — the skill's `description` triggers it automatically.
- **Plan only:** `/gitvibe:plan 47` (or `plan local issue 47`) — writes and refines `.gitvibe/plans/47-plan.md` without touching code. Run `/gitvibe:fix-issue 47` afterwards to implement it.

### Update

After changes are pushed to this repo:

```
/plugin marketplace update gitvibe
```

## OpenAI Codex

Codex reads the **same** `.claude-plugin/` marketplace as Claude Code — it does not need its own marketplace manifest. (Verified with `codex` CLI 0.167.0: `codex plugin add` installs the plugin from `.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`, pulling in `skills/`.)

### Register the marketplace and install

```bash
codex plugin marketplace add singhshashi/gitvibe-plugin
codex plugin add gitvibe@gitvibe
```

`gitvibe@gitvibe` = the plugin named `gitvibe` from the marketplace named `gitvibe`. The plugin updates itself when the `version` in `.claude-plugin/plugin.json` changes. Enablement can be toggled per trusted repo in `.codex/config.toml`:

```toml
[plugins."gitvibe@gitvibe"]
enabled = true
```

Each skill's Codex UI metadata (display name, short description) comes from its `skills/<skill>/agents/openai.yaml`, co-located with the skill.

### Explicit skill invocation

Invoke a skill by name with the `$` prefix:

```text
$fix-issue Fix issue #47.
$plan Plan issue #47.
```

The skill name (`fix-issue`) comes from the `name:` field in `skills/fix-issue/SKILL.md`. Codex discovers skills automatically from the root `skills/` directory — no separate Codex copy exists.

### Differences from Claude Code

- **Explicit-invocation syntax.** Both agents run the same skill; only the way to force it by name differs — Claude Code uses `/gitvibe:fix-issue 47`, Codex uses `$fix-issue Fix issue #47.` Natural-language invocation ("fix local issue 47") is identical on both.
- **Implicit invocation.** The official Codex docs describe no configuration field for disabling implicit (automatic) skill invocation, so none is set here — adding an undocumented field would not be honored. Codex decides when to consider a skill from its `description:`; the `$fix-issue` syntax forces explicit selection. If you need explicit-only behavior, use `$fix-issue` and avoid phrasing that would implicitly match the skill description.

## Shared skill & prerequisites

Regardless of agent, the plugin is installed globally while git-vibe is per-repo, so the workflow begins with a preflight check. If the `git vibe` CLI isn't installed, or the current repo hasn't been initialised with `git vibe init`, the workflow reports that and stops — it won't initialise your repo for you or fall back to another issue tracker.

Plans are written in **GitVibe Plan Compatibility v1** format — a single `gitvibe-plan` JSON block that maps each P-item to files and base-commit line ranges — and validated with `git vibe plan check`. The installed git-vibe must include that command; the preflight stops with a message if it doesn't.

## Layout

```
.claude-plugin/
  marketplace.json   # the ONE marketplace manifest — Claude Code AND Codex read it
  plugin.json        # plugin manifest (name, version, description)
skills/
  fix-issue/         # one directory per skill
    SKILL.md         # shared skill — used by ALL agents
    agents/
      openai.yaml    # Codex/OpenAI display metadata for this skill
    templates/
      fix-plan.md    # required plan structure (also used by plan/)
  plan/              # planning-only skill; writes .gitvibe/plans/<N>-plan.md
    SKILL.md
    agents/
      openai.yaml
```

Adding another agent means adding a `skills/<skill>/agents/<agent>.*` file beside each skill — not another root-level marketplace manifest. Adding another skill means a new `skills/<skill>/` directory.
