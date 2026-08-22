---
name: vanillabp-upgrade-v1-to-v2
description: Upgrade an application from VanillaBP 1 to VanillaBP 2. Surveys a version 1 project for everything that has to change, then applies it. Covers Java 21 and Spring Boot 4 or Quarkus 3.37, the renamed adapter artifacts, the META-INF/workflow-module descriptor, default-adapter becoming prioritized-adapters, adapter settings moving to vanillabp.adapters, the removed ProcessService overloads, @BpmnProcess.primary, and the transaction annotations on workflow services. Use whenever a project depends on VanillaBP 1.x, whenever someone asks about upgrading or migrating to VanillaBP 2, or whenever a build against VanillaBP 2 fails on those symbols.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Upgrading from VanillaBP 1 to VanillaBP 2

The business code stays. `@WorkflowService`, `@WorkflowTask`, `@TaskId`, `@TaskParam`,
`@TaskEvent`, the multi-instance annotations and `ProcessService` keep their meaning, and apart
from three removed overloads no method changed its signature. So do the workflow aggregates,
their persistence and their relation to the workflow, and so do the BPMN models: on Camunda 8
they stay byte-identical, so the upgrade deploys no new process version.

What changes are dependencies, configuration and a handful of behaviors.

Do not confuse this with BPMS migration, which means moving workflows from one engine to
another and is a different topic entirely.

## How to run this

Survey first, report, then change. An upgrade applied blindly hides which of its steps caused a
later surprise.

### Step 0: survey the project

Work through [references/detection.md](references/detection.md). It lists what to grep for and
what each finding means. Produce a written inventory before touching a file:

- the platform and its version, the Java version, the VanillaBP version and the adapter in use,
- every workflow module and how its id is derived today,
- every `vanillabp.*` and BPMS-specific configuration key in every profile and test resource,
- every occurrence of the removed API,
- every transaction annotation reaching a `@WorkflowTask` method,
- every hand-written `ProcessService` or `AggregatePersistenceAware` implementation.

Show that inventory to the user, with the count per item. It is also the checklist you tick off.

### Step 1: lift the runtime

Java 21, Spring Boot 4.1 or Quarkus 3.37. Spring Boot 4 modularized
`spring-boot-autoconfigure`, so some imports of your own code move and the test slices need
explicit dependencies. That is Spring's change and not VanillaBP's, so follow Spring Boot's
own migration guide for it.

### Step 2: swap the dependencies

The adapter artifacts were renamed and every adapter now has a Quarkus artifact too. The table
is in [references/dependencies.md](references/dependencies.md). Then build: the compiler points
straight at the three removed overloads, at `@BpmnProcess.primary` and at the moved
`AggregatePersistenceAware` import.

### Step 3: declare the workflow modules, then move the configuration

Version 2 wants the module id stated explicitly in a `META-INF/workflow-module` descriptor
rather than derived from `spring.application.name` or from a file name. Keep the ids you had,
because they name BPMS tenants and resource directories, and changing one detaches the running
workflows of that module from their configuration.

Then move the configuration key by key, following
[references/configuration.md](references/configuration.md). Start the application afterwards and
follow the startup messages, which name every missing key.

### Step 4: delete the transaction annotations

Remove `@Transactional` from workflow services and check that nothing else wraps a
`@WorkflowTask` method in a transaction of its own. The before and after is in
[references/code-changes.md](references/code-changes.md).

### Step 5: verify against the BPMS

Start one workflow of every process, complete an asynchronous task, correlate a message. On a
remote BPMS also look into the outbox store once, where entries should reach DONE, and decide
the retention you want.

Then read [references/behavior-changes.md](references/behavior-changes.md) with the user. Those
are the things that changed without your code changing, and several of them need a decision
and not an edit.

## Two decisions that are easy to miss

Both are silent in the sense that the application starts and runs, and both change which
workflows it can still find.

**Tenants.** Version 1 isolated a workflow module by a BPMS tenant. Version 2 replaces
`use-tenants` with `name-clash-avoidance`, and both Camunda adapters default to `none`. An
application upgraded from version 1 therefore has to configure `by-adapter` to keep its tenants,
on either BPMS. Camunda 7 users especially: with the default the deployment lands in no tenant,
while the workflows started under version 1 live in their tenants and are found through them.
Set `by-adapter` before starting the upgraded application against an existing database.

**Camunda 7 expressions.** Version 1 answered every BPMN expression by reading the aggregate.
Version 2 writes the shared values as process variables and lets the engine resolve them, and
keeps the live read only as a fallback for workflows that were already running. Version 2.1
removes that fallback. While the application starts, the Camunda 7 adapter lists every
expression needing attention, and that list is the real migration backlog.

## Source

Everything here follows
[Migrating from VanillaBP 1 to 2](https://github.com/vanillabp/adapter-platform-integration/wiki/Migrating-from-version-1),
which is the authoritative page and wins where this skill is out of date. Building or extending
an application is a different job with its own skill, `vanillabp`.
