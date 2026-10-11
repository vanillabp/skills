![VanillaBP](https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/profile/vanillabp-headline.png)

# VanillaBP agent skills

[![Apache License V.2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](./LICENSE)

Skills which teach an AI coding agent how to build business process applications with
[VanillaBP](https://www.vanillabp.io). They follow the
[Agent Skills specification](https://agentskills.io/specification). So the same files work in
Claude Code, GitHub Copilot, VS Code, Cursor, OpenAI Codex, Gemini CLI and the other agents
which support the standard.

| Skill | What it does |
|---|---|
| [`vanillabp`](skills/vanillabp) | Builds a new application from a BPMN model, or adds a BPMN scenario to an existing one. It picks the matching [blueprints](https://github.com/vanillabp-blueprints) and combines them |
| [`vanillabp-upgrade-v1-to-v2`](skills/vanillabp-upgrade-v1-to-v2) | Looks through a VanillaBP 1 project for everything the upgrade to version 2 touches, then applies it |
| [`business-cockpit`](skills/business-cockpit) | Leads a question about the [Business Cockpit](https://github.com/vanillabp/business-cockpit) to the page which answers it: reporting from a workflow module, serving a user task form, running a cockpit of your own, and building a cockpit user interface in any framework |
| [`business-cockpit-upgrade-to-blocking`](skills/business-cockpit-upgrade-to-blocking) | Looks through a customized [Business Cockpit](https://github.com/vanillabp/business-cockpit) application for everything the move from reactive WebFlux to blocking Spring MVC on virtual threads touches, then applies it |

The skills link the reference documentation instead of copying it. So an agent which uses them
reads the current wiki, not an old copy. Where a skill and the documentation disagree, the
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
example `--agent claude-code`, `--agent github-copilot` or `--agent cursor`. Later,
`gh skill update` pulls newer versions, and `gh skill list` shows what is installed.

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

The skills load themselves when a task matches their description. So in most cases there is
nothing to call. Some agents let you name a skill directly. There `/vanillabp`,
`/vanillabp-upgrade-v1-to-v2`, `/business-cockpit` and `/business-cockpit-upgrade-to-blocking`
do that.

Give the agent the BPMN model when the task is about a process. The skills refuse to invent
one. The general skill picks the blueprints it copies from by the element types the model
really contains.

## Where the knowledge comes from

| Source | What it carries |
|---|---|
| [Blueprints](https://github.com/vanillabp-blueprints) | one application you can run per aspect, listed in a catalogue for machines |
| [Platform wiki](https://github.com/vanillabp/adapter-platform-integration/wiki) | concepts, workflow modules, platform integration, configuration |
| [spi-for-java](https://github.com/vanillabp/spi-for-java) | the annotations and interfaces applications use |
| the wiki of each [BPMS adapter](https://github.com/vanillabp/adapter-platform-integration/wiki/BPMS-adapters) | everything specific to one engine |
| [business-cockpit](https://github.com/vanillabp/business-cockpit) | the cockpit container, its extension points, how a customized cockpit is built, and the description a cockpit user interface is built from |

These skills are for people who build applications with VanillaBP. Skills for working on
VanillaBP itself live in
[development-workspace](https://github.com/vanillabp/development-workspace).

## Contributing

Issues and pull requests are welcome here. A change to how VanillaBP itself works belongs in
the repository which documents it. The skill follows afterwards.

Check a change before you open a pull request:

```bash
gh skill publish --dry-run
```

[`CONTRIBUTING.md`](./CONTRIBUTING.md) says the rest: how to try out a skill before you propose
it, and what to ask before a change.

## Noteworthy and contributors

VanillaBP was developed by [Phactum](https://www.phactum.at) with the intention of giving back
to the community as it has benefited from the community in the past.

## License

Copyright 2026 Phactum Softwareentwicklung GmbH

Licensed under the Apache License, Version 2.0.
