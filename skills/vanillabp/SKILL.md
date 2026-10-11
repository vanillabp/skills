---
name: vanillabp
description: Build or extend a business process application with VanillaBP. Turns a BPMN model into a workflow module, grafts BPMN scenarios such as user tasks, message correlation, timers, boundary events, call activities or multi-instance onto an existing application, and picks persistence and BPMS adapter configuration. Use whenever a project uses VanillaBP, whenever a BPMN model has to be implemented in Java, and whenever the terms workflow module, workflow aggregate, @WorkflowService, @WorkflowTask or ProcessService come up.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Building applications with VanillaBP

VanillaBP applies hexagonal architecture to business process applications. A BPMN (Business
Process Model and Notation) model drives the process. Your code implements the tasks in it.
None of that code knows which business process engine (BPMS) runs it. Choosing the BPMS is a
Maven profile, not a code change. That is why nothing you write here may refer to an engine
API.

Two terms carry the whole design:

A *workflow module* bundles the BPMN models of one use case together with the code which
implements them. It is normally a JAR which an application pulls in. The application decides
which BPMS adapter is on the classpath.

A *workflow aggregate* is one stored entity per workflow instance. It holds every piece of
state the process needs. There are no process variables.

## Before you write anything

Find out these four. Ask the user where the project does not answer them:

1. The platform, Spring Boot or Quarkus. Everything after this differs, and the two are never
   mixed in one project.
2. The BPMS: Camunda 7, Camunda 8 or the bpm-crafters Process-Engine-API.
3. Whether this is a new application or an existing one to extend.
4. The BPMN model. Without a model there is nothing to implement. So ask for the file instead
   of inventing a process.

## Procedure

### Step 1: read the catalogue

The blueprints are the reference implementations: one small application you can run, per
aspect. A catalogue for machines lists them. Fetch both files:

- `https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/blueprints.yaml`
- `https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`

The catalogue decides, and it changes as blueprints are added. Where it disagrees with anything
in this skill, the catalogue wins. If you cannot reach the network, work from
[references/catalogue.md](references/catalogue.md), an older copy of the same index. Say that
you did.

### Step 2: parse the BPMN

Collect the element types the model uses (`bpmn:UserTask`, `bpmn:BoundaryEvent`,
`bpmn:MessageEventDefinition` and so on), together with the attributes set on them. This list
selects the blueprints.

### Step 3: pick the blueprints

Compare the element types with `covers.bpmn` in the catalogue, and choose for the target
platform:

- one workflow module structure from `module-*`, starting at `module-single`,
- one persistence from `persistence-*`, JPA unless the user says otherwise,
- every `bpmn-*` scenario the model requires.

You may only use an entry whose `platforms.<platform>.status` is `available`. Somebody works on
a `planned` entry. A `not-applicable` entry names the reason why that platform cannot have it.
So pick a different blueprint instead of porting it yourself.

### Step 4: read the AGENTS.md of each chosen blueprint

Every blueprint repository has one. The catalogue links it as `platforms.<platform>.agents_md`.
It names the placeholders, the files which carry the aspect, and the steps to add it to an
existing project.

### Step 5: apply the deltas

`module-single` is the base, and every `bpmn-*` blueprint is a delta on top of it. They all
have the same structure. So several deltas combine into one application.

For a new application, clone the delivered repository of the base blueprint for your platform
(`https://github.com/vanillabp-blueprints/<blueprint-id>-<platform>`). Replace the
placeholders, all four of them, consistently. For an existing application, follow the section
"Adding this blueprint to an existing project" in each blueprint's AGENTS.md. Keep the
structure the project already has.

[references/project-structure.md](references/project-structure.md) has the structure every
blueprint uses, the placeholders and the layout of the configuration. Read it before you write
the first class. [references/rules.md](references/rules.md) has the rules which hold for every
blueprint. If you break one of them, a VanillaBP application stops working without a word.

### Step 6: verify by running it

```bash
mvn verify
```

Every blueprint ships an integration test which plays through its aspect. Keep that test and
adapt it to the use case. Generated code which never ran is a guess.

### Step 7: fix what startup validation reports

VanillaBP checks the wiring between BPMN and code while the application boots. On Quarkus it
already checks while the application builds. Its messages name the fix. Never work around such
a message. It describes a real inconsistency. Follow the message step by step until the
configuration is complete.

If you cannot place a message, look in [references/symptoms.md](references/symptoms.md). It
leads from what you see to the section which explains it.

Steps 6 and 7 are the only reason to trust the result. Do not skip them.

## Operating the result

Blueprints offer GET endpoints only, so you can walk through a process in a browser with no
tooling at all. Keep that property in what you generate. At every wait state, log one complete,
clickable URL for each possible next step.

```
Accept -> http://localhost:8080/api/loan-approval/{id}/assess-risk/{taskId}?riskIsAcceptable=true
Deny   -> http://localhost:8080/api/loan-approval/{id}/assess-risk/{taskId}?riskIsAcceptable=false
```

Those logged URLs are also the cheapest way for you to see which state a workflow reached.

## When something is not covered

Blueprints cover only what the SPI (service provider interface) and the platform integrations
provide, on purpose. If a task cannot be solved without details of a BPMS, say so instead of
inventing configuration. [references/documentation-map.md](references/documentation-map.md)
says where each topic is documented.

## When something goes wrong

Start at [references/symptoms.md](references/symptoms.md). It lists the messages and exceptions
which come up around VanillaBP. It says what each one is and links the section which answers
it. Search it with the words you see before you search the wikis. Add a row whenever you find
an answer which was not there.

Where the application reports into a VanillaBP Business Cockpit, search
[references/business-cockpit-symptoms.md](references/business-cockpit-symptoms.md) as well. It
is the same kind of list, written for the cockpit server and for the adapters which report into
it. For work on the cockpit side itself, not only reading an error from it, use the skill
`business-cockpit`.

Upgrading an application from VanillaBP 1 is a different job with its own skill,
`vanillabp-upgrade-v1-to-v2`.
