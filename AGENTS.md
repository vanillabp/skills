# Working on the VanillaBP skills

This file is for agents which change this repository. What each skill does, how it is installed
and where its knowledge comes from is in [`README.md`](./README.md). How a change is checked and
what is asked before a change is in [`CONTRIBUTING.md`](./CONTRIBUTING.md). Read both before you
change anything.

The skills below `skills/` are instructions for other agents. When you rewrite one, keep the
meaning of every instruction: every must and never, every condition, every step and its order.
Plainer, never weaker or stronger.

## How we write

Most people who read this repository read English as a second language, and so does the
maintainer. Long sentences, rare words and stacked nouns slow them down. Write so that
nobody has to read a sentence twice.

Short main sentences, one thought each. One subordinate clause is enough. Active voice.
The common word instead of the rare one: `use` instead of `leverage`, `about` instead of
`regarding`, `so` instead of `consequently`, `has` instead of `possesses`. A technical term
stays a technical term, but say what it means the first time it turns up, and write an
abbreviation out once. If a sentence trips you up when you read it aloud, rewrite it.

This holds for every English text here: the `SKILL.md` files and the pages below `references/`,
the README files, `CONTRIBUTING.md`, this file, and the texts of commits and pull requests. This
repository has no wiki of its own. If it gets one, the rule holds there as well, because a wiki
clone has no `AGENTS.md` of its own.

Nothing a program reads is renamed for the sake of language. The name of a skill, the keys of
its front matter, file names and the anchors other pages link to stay as they are, because
installations, the registry and other pages point at them. The `description` of a skill decides
whether an agent loads it, so a change to its wording is a change to behaviour.

Before and after, from this repository:

> Survey first, report, then change. An upgrade applied blindly hides which of its steps caused a
> later surprise.

becomes

> Look first, report, then change. If you apply an upgrade blindly, you cannot tell later which of
> its steps caused a surprise.

That one is from the skill `vanillabp-upgrade-v1-to-v2`. The next is from [`README.md`](./README.md):

> The skills load themselves when a task matches their description, so in most cases there is
> nothing to invoke. Where an agent supports naming a skill directly, `/vanillabp`,
> `/vanillabp-upgrade-v1-to-v2`, `/business-cockpit` and `/business-cockpit-upgrade-to-blocking`
> do that.

becomes

> The skills load themselves when a task matches their description. So in most cases there is
> nothing to call. Some agents let you name a skill directly. There `/vanillabp`,
> `/vanillabp-upgrade-v1-to-v2`, `/business-cockpit` and `/business-cockpit-upgrade-to-blocking`
> do that.
