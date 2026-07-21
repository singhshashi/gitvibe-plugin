# gitvibe — Claude Code plugin

A [Claude Code](https://claude.com/claude-code) plugin for the **git-vibe** local issue tracker. It bundles:

- the **`gitvibe`** skill — an investigate → plan → fix workflow, with a required `fix-plan` template
- the **`/gitvibe:fix-issue`** command — a thin wrapper that runs that workflow for a given issue number

This repository is both the plugin and its marketplace.

## Install

```
/plugin marketplace add singhshashi/gitvibe-plugin
/plugin install gitvibe@gitvibe
```

`gitvibe@gitvibe` = the plugin named `gitvibe` from the marketplace named `gitvibe`.

## Usage

- `/gitvibe:fix-issue 47` — investigate and fix local issue #47
- Or trigger the skill by natural language: "fix local issue 47"

### Requirements

The plugin is installed globally but git-vibe is per-repo, so both commands start with a preflight check. If the `git vibe` CLI isn't installed, or the current repo hasn't been initialised with `git vibe init`, the workflow reports that and stops — it won't initialise your repo for you or fall back to another issue tracker.

## Update

After changes are pushed to this repo:

```
/plugin marketplace update gitvibe
```

## Layout

```
.claude-plugin/
  marketplace.json   # marketplace manifest (plugin source: ".")
  plugin.json        # plugin manifest
commands/
  fix-issue.md       # /gitvibe:fix-issue
skills/
  gitvibe/
    SKILL.md
    templates/
      fix-plan.md    # required plan structure
```
