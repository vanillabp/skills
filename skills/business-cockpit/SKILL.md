---
name: business-cockpit
description: Work with the VanillaBP Business Cockpit, the application which shows the cases and user tasks a workflow module reports. Covers connecting a workflow module and reporting into it, the details providers which fill a list column, serving a user task form, running the cockpit server, who sees which case, and where to start when somebody wants a cockpit of their own. Use whenever a task mentions the Business Cockpit, a task list or case list for business users, vanillabp.cockpit configuration keys, a @UserTaskDetailsProvider or @WorkflowDetailsProvider method, or a user task form.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Working with the VanillaBP Business Cockpit

The Business Cockpit is the window a business user looks through. A workflow module reports
what happens in its processes. The cockpit server stores those reports. A user interface shows
two lists: the cases, and the user tasks somebody still has to work on. A click on a task opens
a form which the workflow module itself serves.

This skill is a map, not a manual. Everything the cockpit does is documented in the repository
which holds it. This page says which document answers which question. Where this page and that
documentation disagree, the documentation is right.

The cockpit has three sides, and nearly every question belongs to exactly one of them:

| Side | What it is | Where it is documented |
|---|---|---|
| the workflow module | your application, reporting cases and tasks | [Connecting a workflow module](https://github.com/vanillabp/business-cockpit/wiki/Connecting-a-workflow-module) |
| the cockpit server | the application storing the reports and answering the GUI API | [Running the Business Cockpit](https://github.com/vanillabp/business-cockpit/wiki/Running-the-Business-Cockpit) |
| the user interface | what the browser runs | the description named below |

## Reporting from a workflow module

One dependency and a few configuration keys connect a module to a cockpit. The module then
reports on its own. Your business code only has to fill in what a list column should show.

- [Connecting a workflow module](https://github.com/vanillabp/business-cockpit/wiki/Connecting-a-workflow-module)
  is the first page: the dependency, how a module registers itself, and whether the reports
  travel over REST or Kafka.
- [Reporting workflows and user tasks](https://github.com/vanillabp/business-cockpit/wiki/Reporting-workflows-and-user-tasks)
  is the page you come back to: the provider methods, the business data which fills a column,
  who may see a module, and how to report a change your own code made.
- [Templates](https://github.com/vanillabp/business-cockpit/wiki/Templates) renders the title and
  the search text of a task, per language.
- [Configuration](https://github.com/vanillabp/business-cockpit/wiki/Configuration) lists every key
  of both sides and the four levels a value may be written at.

## Serving a form for a user task

A user task in the cockpit is a row in a list until somebody opens it. What they then see comes
out of the workflow module. Module federation loads it into the cockpit's page.

- [User task forms and status sites](https://github.com/vanillabp/business-cockpit/wiki/User-task-forms-and-status-sites)
  is this contract from the module's side: what it offers, how it builds, and the columns it
  adds to the two lists.
- [Developing UI components locally](https://github.com/vanillabp/business-cockpit/wiki/Developing-UI-components-locally)
  is the dev shell. With it you can write a form without starting a cockpit, a MongoDB and a
  BPMS (business process management system).

## Running and customizing the cockpit itself

- [Architecture](https://github.com/vanillabp/business-cockpit/wiki/Architecture) explains why
  the store is a document database, and which path a report takes. Read it before you change
  anything.
- [Security](https://github.com/vanillabp/business-cockpit/wiki/Security) answers who sees which
  case and which task, how a user reaches a workflow module, and how an identity provider goes in
  front.
- [Building a custom Business Cockpit](https://github.com/vanillabp/business-cockpit/wiki/Building-a-custom-Business-Cockpit)
  is for a cockpit of your own: the library next to the container, and the beans nobody can
  write for you.

## Building a cockpit of your own

Every organisation brings its own web framework, its own component library and its own design.
So the cockpit delivers no user interface. A skill generates one instead: the skill
`business-cockpit-creator`. It lives in the `business-cockpit` repository, next to the interfaces
it describes, because a copy anywhere else would go out of date.

The skill first asks the user about framework, design, languages, screens, lists and login. Then
it generates two repositories: the user interface with its dev shells, and the backend
application with a local environment to run it. At the end it checks both against a checklist.

Install it next to this one:

```bash
gh skill install vanillabp/business-cockpit business-cockpit-creator
```

Load that skill when the task is a new cockpit, its user interface or its backend application. Do
not work from this page for that job.

[Customizing the user interface](https://github.com/vanillabp/business-cockpit/wiki/Customizing-the-user-interface)
is the same subject in the wiki: the packages, changing the look, and what a user interface has to
answer.

## When something goes wrong

Search
[the symptom list](https://github.com/vanillabp/skills/blob/main/skills/vanillabp/references/business-cockpit-symptoms.md)
with the words you see. It leads from the messages and exceptions which come up around the
cockpit to the section which explains them, including the ones in the browser console. Some
rows are about something you do not see at all, such as a task which never turns up in a list.
They name the DEBUG line which says why.

The list lives next to the `vanillabp` skill, because it was written before this one existed.
It covers both sides of the cockpit.

## Two neighbouring jobs with their own skills

To build the application whose processes the cockpit shows, use the skill `vanillabp`. To move
a customized cockpit from the reactive container to the blocking one, use
`business-cockpit-upgrade-to-blocking`. An application which comes from version 1 of the
cockpit adapters reads
[Migrating from version 1](https://github.com/vanillabp/business-cockpit/wiki/Migrating-from-version-1)
first.
