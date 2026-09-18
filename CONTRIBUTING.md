# Contributing

This repository holds the agent skills for building applications with
[VanillaBP](https://www.vanillabp.io). They follow the
[Agent Skills specification](https://agentskills.io/specification), so one skill is a directory
below `skills/` with a `SKILL.md` and the reference pages it links. The skills are for people
building applications. Skills for working on VanillaBP itself live in
[development-workspace](https://github.com/vanillabp/development-workspace).

There is no Java here and no `AGENTS.md`. [`README.md`](./README.md) says what each skill does, how
it is installed and where its knowledge comes from, and it is the first thing to read.

A skill links the documentation instead of copying it, which is the rule this repository stands on.
An agent reading a skill should end up in the current wiki, not in a snapshot of it from the day the
skill was written. Where a skill and the documentation disagree, the documentation is right and the
skill is the defect.

## Changing a skill and checking it

A change is a text change: `SKILL.md`, a page below `references/`, or both. The front matter carries
the name, the description an agent matches a task against, and the license. The description is what
decides whether a skill is loaded at all, so a change to it is a change to behaviour.

Check it before you open a pull request:

```bash
gh skill publish --dry-run
```

That needs `gh` 2.90 or later, and it reports what a publish would do without doing it. There is no
build and no continuous integration in this repository, so this command and your own reading are the
whole check.

Then try the skill. Install it into a scratch project the way [`README.md`](./README.md) describes,
give the agent a task the skill is meant for, and see whether it loads by itself. A skill which only
works when it is named by hand has a description problem.

## How we write

Most people who read this repository read English as a second language, and so does the maintainer.
Long sentences, rare words and stacked nouns slow them down. Write so that nobody has to read a
sentence twice.

Short main sentences, one thought each. One subordinate clause is enough. Active voice. The common
word instead of the rare one: `use` instead of `leverage`, `about` instead of `regarding`, `so`
instead of `consequently`. A technical term stays a technical term, but say what it means the first
time it turns up, and write an abbreviation out once. If a sentence trips you up when you read it
aloud, rewrite it.

An agent reads this text as well as a person, and both of them fall for the same thing: a sentence
which sounds right and says nothing. Nothing a program reads is renamed for the sake of language,
the name of a skill included, because installations and the registry point at it.

## What is asked before a change

A skill describes how VanillaBP works. Where your change says something new about that, the change
belongs into the repository which documents it, the wiki or the blueprints, and the skill follows
afterwards. A skill which teaches a rule nothing else states is where the two start to drift, and
the skill is the copy which goes stale.

A new skill is the second question. Its name is what installations and the registry point at, and
its description decides which tasks reach it, so ask before you add one rather than after.

## Opening a pull request

Work on a branch of your own and keep one subject per pull request. The description says what moved
and why it had to, and it is worth naming the agent you tried the change with, because that is the
only evidence there is here.

## License

VanillaBP is published under the [Apache License, Version 2.0](./LICENSE), and by contributing you
agree that your contribution is licensed the same way. [`NOTICE`](./NOTICE) names who holds the
copyright, and every `SKILL.md` carries the same license in its front matter.
