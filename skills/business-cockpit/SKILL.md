---
name: business-cockpit
description: Work with the VanillaBP Business Cockpit, the application which shows the cases and user tasks a workflow module reports. Covers connecting a workflow module and reporting into it, the details providers which fill a list column, serving a user task form through module federation, running the cockpit server, who sees which case, building a cockpit application of your own, and building a cockpit user interface in any web framework. Use whenever a task mentions the Business Cockpit, a task list or case list for business users, vanillabp.cockpit configuration keys, a @UserTaskDetailsProvider or @WorkflowDetailsProvider method, a user task form, or replacing the delivered React front end.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Working with the VanillaBP Business Cockpit

The Business Cockpit is the window a business user looks through. A workflow module reports what
happens in its processes, the cockpit server stores those reports, and a user interface shows two
lists: the cases and the user tasks somebody still has to work on. Clicking a task opens a form
which the workflow module itself serves.

This skill is a map, not a manual. Everything the cockpit does is documented in the repository
which holds it, and this page says which document answers which question. Where this page and that
documentation disagree, the documentation is right.

The cockpit has three sides, and nearly every question belongs to exactly one of them:

| Side | What it is | Where it is documented |
|---|---|---|
| the workflow module | your application, reporting cases and tasks | [Connecting a workflow module](https://github.com/vanillabp/business-cockpit/wiki/Connecting-a-workflow-module) |
| the cockpit server | the application storing the reports and answering the GUI API | [Running the Business Cockpit](https://github.com/vanillabp/business-cockpit/wiki/Running-the-Business-Cockpit) |
| the user interface | what the browser runs | the description named below |

## Reporting from a workflow module

One dependency and a handful of configuration keys connect a module to a cockpit. The module then
reports on its own, and your business code only has to fill in what a list column should show.

- [Connecting a workflow module](https://github.com/vanillabp/business-cockpit/wiki/Connecting-a-workflow-module)
  is the first page: the dependency, how a module registers itself, and whether the reports travel
  over REST or Kafka.
- [Reporting workflows and user tasks](https://github.com/vanillabp/business-cockpit/wiki/Reporting-workflows-and-user-tasks)
  is the one you come back to: the provider methods, the business data a column is filled from, who
  may see a module, and reporting a change your own code made.
- [Templates](https://github.com/vanillabp/business-cockpit/wiki/Templates) renders the title and
  the search text of a task, per language.
- [Configuration](https://github.com/vanillabp/business-cockpit/wiki/Configuration) lists every key
  of both sides and the four levels a value may be written at.

## Serving a form for a user task

A user task in the cockpit is a row in a list until somebody opens it. What they then see comes out
of the workflow module, loaded into the cockpit's page through module federation.

- [User task forms and status sites](https://github.com/vanillabp/business-cockpit/wiki/User-task-forms-and-status-sites)
  is this contract from the module's side: what it exposes, how it builds, and the columns it
  contributes to the two lists.
- [Developing UI components locally](https://github.com/vanillabp/business-cockpit/wiki/Developing-UI-components-locally)
  is the dev shell, so a form can be written without starting a cockpit, a MongoDB and a BPMS.

## Running and customizing the cockpit itself

- [Architecture](https://github.com/vanillabp/business-cockpit/wiki/Architecture) explains why the
  store is a document database and which path a report takes. Read it before changing anything.
- [Security](https://github.com/vanillabp/business-cockpit/wiki/Security) answers who sees which
  case and which task, how a user reaches a workflow module, and how an identity provider goes in
  front.
- [Building a custom Business Cockpit](https://github.com/vanillabp/business-cockpit/wiki/Building-a-custom-Business-Cockpit)
  is for a cockpit of your own: the library next to the container, and the beans nobody can write
  for you.

## Building a user interface

The cockpit ships a user interface written in React. It is the worked example, not the product.
Every organisation brings its own framework, its own version of it, its own component library and
its own design, so the delivered application gets copied and rebuilt, and the copy then drifts away
from what the cockpit expects.

So the user interface is described rather than delivered. The description lives in the
`business-cockpit` repository, next to the interfaces it describes, and it is complete enough to
build a front end from without reading the React sources:
[skills/business-cockpit-user-interface](https://github.com/vanillabp/business-cockpit/tree/main/skills/business-cockpit-user-interface).

It is a skill of its own, so load it when the task is a user interface. Do not work from this page
for that job, and do not read the React sources instead: they are one way of answering the
description, and it is the description that the cockpit holds a user interface to. What you find
there:

| Page | What it answers |
|---|---|
| `SKILL.md` | what to build, in which order, what a cockpit user interface never does, and how it gets served |
| `references/gui-api.md` | logging in, the endpoints, paging, sorting and filtering over reported business data, the suggestions under a search field, the modes of a list, and the live update stream |
| `references/module-federation.md` | the four parts of a workflow module, how one is loaded, and what the host has to share |
| `references/types.md` | the types, and which of them are the contract and which are convenience |
| `references/behaviour.md` | the screens, the columns, the cells, what a list does while somebody watches it, and the actions on a task |
| `references/dev-shell.md` | the workbench for a form, and the simulator behind it |
| `references/rough-edges.md` | the seventeen places where the obvious thing is the wrong thing |

Two things to know before you open it. The text is German until it has been reviewed, and it will
be translated afterwards; its front matter is English, because that is what an agent matches a task
against. And it names its own gaps: where nobody could say what a field is for, it says so instead
of guessing. Treat a named gap as a question for the maintainers, not as something to fill in with
a plausible answer.

[Customizing the user interface](https://github.com/vanillabp/business-cockpit/wiki/Customizing-the-user-interface)
is the same subject for a reader who is not building one: what the delivered application gets you,
how far a copy of it goes, and how the description is meant to be used.

## When something goes wrong

Search
[the symptom list](https://github.com/vanillabp/skills/blob/main/skills/vanillabp/references/business-cockpit-symptoms.md)
with the words you see. It maps the messages and exceptions which come up around the cockpit onto
the section which explains them, including the ones which show up in the browser console. Some rows
are about something you do not see at all, such as a task which never turns up in a list, and name
the DEBUG line which says why.

The list lives next to the `vanillabp` skill because it was written before this one existed. It
covers both sides of the cockpit.

## Two neighbouring jobs with their own skills

Building the application whose processes the cockpit shows is the skill `vanillabp`. Moving a
customized cockpit from the reactive container to the blocking one is
`business-cockpit-upgrade-to-blocking`. An application coming from version 1 of the cockpit adapters
reads
[Migrating from version 1](https://github.com/vanillabp/business-cockpit/wiki/Migrating-from-version-1)
first.
