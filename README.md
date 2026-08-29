![VanillaBP](https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/profile/vanillabp-headline.png)

# VanillaBP agent skills

Skills teaching an AI coding agent how to build business process applications with
[VanillaBP](https://www.vanillabp.io). They follow the
[Agent Skills specification](https://agentskills.io/specification), so the same files work in
Claude Code, GitHub Copilot, VS Code, Cursor, OpenAI Codex, Gemini CLI and the other agents
supporting the standard.

| Skill | What it does |
|---|---|
| [`vanillabp`](skills/vanillabp) | Builds a new application from a BPMN model, or grafts a BPMN scenario onto an existing one, by picking and composing the matching [blueprints](https://github.com/vanillabp-blueprints) |
| [`vanillabp-upgrade-v1-to-v2`](skills/vanillabp-upgrade-v1-to-v2) | Surveys a VanillaBP 1 project for everything the upgrade to version 2 touches, then applies it |
| [`vanillabp-adapter-building`](skills/vanillabp-adapter-building) | Builds or extends an adapter for a BPMS VanillaBP does not support yet — module layout, the adapter SPI, registration on both platforms |

The first two are for applications; the third is for a team teaching VanillaBP to talk to another
BPMS. All of them link the reference documentation instead of copying it, so an agent using them
reads the current wiki instead of a snapshot. Where a skill and the documentation disagree, the
documentation wins.

## Installing

### With the GitHub CLI

Needs `gh` 2.90 or later.

```bash
gh skill install vanillabp/skills --all               # into the current project
gh skill install vanillabp/skills --all --scope user  # for every project
gh skill install vanillabp/skills vanillabp           # just one of them
```

`gh skill install` picks the directory your agent reads. Name it explicitly with `--agent`, for
instance `--agent claude-code`, `--agent github-copilot` or `--agent cursor`. Later on,
`gh skill update` pulls newer versions and `gh skill list` shows what is installed.

### With the skills registry

```bash
npx skills add vanillabp/skills
```

### By hand

Copy the directories below `skills/` into the place your agent reads. The usual ones are:

| Agent | Project scope | User scope |
|---|---|---|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| GitHub Copilot, VS Code | `.github/skills/` or `.agents/skills/` | `~/.copilot/skills/` |
| Cursor | `.cursor/skills/` | |
| OpenAI Codex | `.codex/skills/` or `.agents/skills/` | `~/.codex/skills/` |
| Several agents together | `.agents/skills/` | `~/.agents/skills/` |

## Using them

The skills load themselves when a task matches their description, so in most cases there is
nothing to invoke. Where an agent supports naming a skill directly, `/vanillabp` and
`/vanillabp-upgrade-v1-to-v2` do that.

Give the agent the BPMN model. Both skills refuse to invent a process, and the general skill
selects the blueprints it copies from by the element types the model actually contains.

## Where the knowledge comes from

| Source | What it carries |
|---|---|
| [Blueprints](https://github.com/vanillabp-blueprints) | one runnable application per aspect, indexed in a machine readable catalogue |
| [Platform wiki](https://github.com/vanillabp/adapter-platform-integration/wiki) | concepts, workflow modules, platform integration, configuration |
| [spi-for-java](https://github.com/vanillabp/spi-for-java) | the annotations and interfaces applications use |
| the wiki of each [BPMS adapter](https://github.com/vanillabp/adapter-platform-integration/wiki/BPMS-adapters) | everything specific to one engine |

These skills are for people building applications with VanillaBP. Skills for contributing to
VanillaBP itself live in
[development-workspace](https://github.com/vanillabp/development-workspace).

## Contributing

Issues and pull requests are welcome here. A change to how VanillaBP itself works belongs into
the repository documenting it, and a skill follows afterwards.

Validate a change before opening a pull request:

```bash
gh skill publish --dry-run
```

## Noteworthy and contributors

VanillaBP was developed by [Phactum](https://www.phactum.at) with the intention of giving back
to the community as it has benefited from the community in the past.

## License

Copyright 2026 Phactum Softwareentwicklung GmbH

Licensed under the Apache License, Version 2.0.
