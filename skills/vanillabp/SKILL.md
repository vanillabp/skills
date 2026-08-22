---
name: vanillabp
description: Build or extend a business process application with VanillaBP. Turns a BPMN model into a workflow module, grafts BPMN scenarios such as user tasks, message correlation, timers, boundary events, call activities or multi-instance onto an existing application, and picks persistence and BPMS adapter configuration. Use whenever a project uses VanillaBP, whenever a BPMN model has to be implemented in Java, and whenever the terms workflow module, workflow aggregate, @WorkflowService, @WorkflowTask or ProcessService come up.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Building applications with VanillaBP

VanillaBP applies hexagonal architecture to business process applications. A BPMN model drives
the process, your code implements the tasks it contains, and none of that code knows which
business process engine (BPMS) executes it. Choosing the BPMS is a Maven profile rather than a
code change, which is why nothing you write here may reference an engine API.

Two terms carry the whole design:

A *workflow module* bundles the BPMN models of one use case together with the code implementing
them. It is normally a JAR that an application pulls in, and the application decides which BPMS
adapter is on the classpath.

A *workflow aggregate* is one persistent entity per workflow instance holding every piece of
state the process needs. There are no process variables.

## Before you write anything

Establish these four, asking the user where the project does not answer them:

1. The platform, Spring Boot or Quarkus. Everything downstream differs, and the two are never
   mixed in one project.
2. The BPMS: Camunda 7, Camunda 8 or the bpm-crafters Process-Engine-API.
3. Whether this is a new application or an existing one to extend.
4. The BPMN model. Without a model there is nothing to implement, so ask for the file rather
   than inventing a process.

## Procedure

### Step 1: read the catalogue

The blueprints are the reference implementations, one small runnable application per aspect.
They are indexed in a machine readable catalogue. Fetch both files:

- `https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/blueprints.yaml`
- `https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`

The catalogue is the authority and it changes as blueprints are added. Where it disagrees with
anything in this skill, the catalogue wins. If you cannot reach the network, work from
[references/catalogue.md](references/catalogue.md), a snapshot of the same index, and say that
you did.

### Step 2: parse the BPMN

Collect the element types the model uses (`bpmn:UserTask`, `bpmn:BoundaryEvent`,
`bpmn:MessageEventDefinition` and so on) together with the attributes set on them. This list is
what selects the blueprints.

### Step 3: pick the blueprints

Compare the element types against `covers.bpmn` in the catalogue and choose for the target
platform:

- one workflow module structure from `module-*`, starting at `module-single`,
- one persistence from `persistence-*`, JPA unless the user says otherwise,
- every `bpmn-*` scenario the model requires.

Only an entry whose `platforms.<platform>.status` is `available` may be used. A `planned` entry
is being worked on. A `not-applicable` entry names the reason that platform cannot have it, so
pick a different blueprint instead of porting it yourself.

### Step 4: read the AGENTS.md of each chosen blueprint

Every blueprint repository carries one, linked from the catalogue as
`platforms.<platform>.agents_md`. It names the placeholders, the files carrying the aspect and
the steps to graft it onto an existing project.

### Step 5: apply the deltas

`module-single` is the base and every `bpmn-*` blueprint is a delta on top of it. They are
structurally identical, so several deltas compose into one application.

For a new application, clone the delivered repository of the base blueprint for your platform
(`https://github.com/vanillabp-blueprints/<blueprint-id>-<platform>`) and replace the
placeholders, all four of them, consistently. For an existing application, follow the
"Adding this blueprint to an existing project" section of each blueprint's AGENTS.md and keep
the structure the project already has.

The structure every blueprint uses, the placeholders and the configuration layout are in
[references/project-structure.md](references/project-structure.md). Read it before writing the
first class. The rules that hold regardless of blueprint are in
[references/rules.md](references/rules.md), and breaking one of them is how a VanillaBP
application quietly stops working.

### Step 6: verify by running it

```bash
mvn verify
```

Every blueprint ships an integration test playing through its aspect. Keep that test and adapt
it to the use case. Generated code that was never executed is a guess.

### Step 7: fix what startup validation reports

VanillaBP checks the wiring between BPMN and code while the application boots, on Quarkus
already while it builds, and its messages name the remedy. Never work around such a message.
It describes a real inconsistency, and the message is meant to be followed step by step until
the configuration is complete.

Steps 6 and 7 are the only reason to trust the result. Do not skip them.

## Operating the result

Blueprints expose GET endpoints only, so a process can be walked through in a browser with no
tooling at all. Keep that property in what you generate: at every wait state log one fully
populated, clickable URL per possible continuation.

```
Accept -> http://localhost:8080/api/loan-approval/{id}/assess-risk/{taskId}?riskIsAcceptable=true
Deny   -> http://localhost:8080/api/loan-approval/{id}/assess-risk/{taskId}?riskIsAcceptable=false
```

Those logged URLs are also the cheapest way for you to see which state a workflow reached.

## When something is not covered

Blueprints deliberately cover only what the SPI and the platform integrations provide. If a
task cannot be solved without BPMS specifics, say so instead of inventing configuration. The
map of where each topic is documented is in
[references/documentation-map.md](references/documentation-map.md).

An application being upgraded from VanillaBP 1 is a different job with its own skill,
`vanillabp-upgrade-v1-to-v2`.
