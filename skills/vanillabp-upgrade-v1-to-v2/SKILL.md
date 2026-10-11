---
name: vanillabp-upgrade-v1-to-v2
description: Upgrade an application from VanillaBP 1 to VanillaBP 2. Surveys a version 1 project for everything that has to change, then applies it. Covers Java 21 and Spring Boot 4 or Quarkus 3.37, the renamed adapter artifacts, the META-INF/workflow-module descriptor, default-adapter becoming prioritized-adapters, adapter settings moving to vanillabp.adapters, the removed ProcessService overloads, @BpmnProcess.primary, and the transaction annotations on workflow services. Also covers the first start refusing an aggregate which shares everything, workflow services being found because they are beans, the version attribute of @BpmnProcess taking effect, decision tables deployed with their workflow module, the Camunda 8 wait for its cluster, and the Camunda 7 serialization format a nested value needs. Use whenever a project depends on VanillaBP 1.x, whenever someone asks about upgrading or migrating to VanillaBP 2, or whenever a build against VanillaBP 2 fails on those symbols.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Upgrading from VanillaBP 1 to VanillaBP 2

The business code stays. `@WorkflowService`, `@WorkflowTask`, `@TaskId`, `@TaskParam`,
`@TaskEvent`, the multi-instance annotations and `ProcessService` keep their meaning. Apart
from three removed overloads, no method changed its signature. The workflow aggregates, their
persistence and their relation to the workflow stay too. So do the BPMN models. On Camunda 8
they stay identical byte for byte, so the upgrade deploys no new process version.

What changes are dependencies, configuration and a few behaviors.

Do not confuse this with BPMS migration. That means moving workflows from one engine to
another, and it is a different topic entirely.

## How to run this

Look first, report, then change. If you apply an upgrade blindly, you cannot tell later which
of its steps caused a surprise.

### Step 0: survey the project

Work through [references/detection.md](references/detection.md). It lists what to grep for and
what each finding means. Write down a list of what you found before you touch a file:

- the platform and its version, the Java version, the VanillaBP version and the adapter in use,
- every workflow module and how its id is derived today,
- every `.dmn` file below a module's resources and how it is deployed today,
- every `@WorkflowService` class and whether it is a bean,
- every `vanillabp.*` configuration key, and every key specific to a BPMS, in every profile and
  test resource,
- every occurrence of the removed API,
- every `@TaskParam` and its declared type, against the value the model maps into it,
- every transaction annotation which reaches a `@WorkflowTask` method,
- every hand-written `ProcessService` implementation, test doubles included.

Show that list to the user, with the count per item. It is also the checklist you tick off.

### Step 1: lift the runtime

Java 21, Spring Boot 4.1 or Quarkus 3.37. Spring Boot 4 split `spring-boot-autoconfigure` into
modules. So some imports of your own code move, and the test slices need explicit dependencies.
That is Spring's change, not VanillaBP's. Follow Spring Boot's own migration guide for it.

### Step 2: swap the dependencies

The adapter artifacts were renamed, and every adapter now has a Quarkus artifact too. The table
is in [references/dependencies.md](references/dependencies.md). On Camunda 8 the version also
names the minor version of the cluster you talk to. So decide which one that is before you
write the coordinate. Then build. The compiler points straight at the three removed overloads
and at `@BpmnProcess.primary`. Persistence needs nothing here. What version 1 could do, JPA and
MongoDB through Spring Data, version 2 does out of the box.

### Step 3: declare the workflow modules, then move the configuration

Version 2 wants the module id stated explicitly in a `META-INF/workflow-module` descriptor.
Version 1 derived it from `spring.application.name`, from the name of the module's
configuration file, or from a static bean method which returns a `WorkflowModuleProperties`.
That type is gone. So find every one of the three and replace it with the descriptor. Keep the
ids you had. They name BPMS tenants and resource directories. If you change one, the running
workflows of that module lose their configuration.

Then move the configuration key by key, following
[references/configuration.md](references/configuration.md). Start the application afterwards
and follow the startup messages. They name every missing key.

One of those messages ends the start instead of guiding you through it, and it is written for
exactly a version 1 application. An aggregate which holds nothing back shares every attribute
it reaches. So version 2 refuses such a workflow until the aggregate says what the models need,
or the workflow allows the full sync. Decide that with the user, because it is about data which
leaves the application. [references/behavior-changes.md](references/behavior-changes.md) has
both ways out.

### Step 4: delete the transaction annotations

Remove `@Transactional` from workflow services. Check that nothing else wraps a `@WorkflowTask`
method in a transaction of its own. [references/code-changes.md](references/code-changes.md)
shows before and after.

Before you delete an annotation, check whether the class is also the business service. Small
projects often keep both in one class, and the business methods still need their transaction.
There the annotation stays, or the class gets split.

### Step 5: verify against the BPMS

Start one workflow of every process, complete an asynchronous task, correlate a message. On a
remote BPMS also look into the outbox store once. Entries there should reach DONE. Decide how
long you want to keep them.

Then read [references/behavior-changes.md](references/behavior-changes.md) with the user. Those
are the things which changed although your code did not. Several of them need a decision, not
an edit.

## Two decisions that are easy to miss

Both are silent: the application starts and runs. And both change which workflows it can still
find.

**Tenants.** Version 1 kept a workflow module apart by a BPMS tenant. Version 2 replaces
`use-tenants` with `name-clash-avoidance`. Its default is `by-adapter` on both Camunda
adapters, and that is what version 1 did: a tenant per workflow module, named after the module.
So an application which ran version 1 with the defaults needs no setting at all. Its workflows
are found where they have always been.

Ask the one question which does need an answer: did the application set `use-tenants: false`?
If it did, its workflows carry no tenant, while the default deploys into one. Then it will miss
every workflow it started. Such an application has to configure `name-clash-avoidance: none`.
On Camunda 8 there is a second case of the same kind: a cluster without multi-tenancy, which
rejects a tenant id outright. The boot ends there with a message which names both ways out, so
you cannot miss it.

**Camunda 7 expressions.** Version 1 answered every BPMN expression by reading the aggregate.
Version 2 writes the shared values as process variables and lets the engine resolve them. It
keeps reading the aggregate directly only as a fallback for workflows which were already
running. The fallback answers a top-level name and nothing else. So it does not cover
`${order.customer.name}`. While the application starts, the Camunda 7 adapter names the
expressions whose value the engine does not hold. While it runs, it reports every name it
answered from the aggregate, and counts the workflows which still depend on one. Watch that
count. It falls on its own as those workflows reach their next sync point. The fallback is
meant to go away once nobody depends on it. Ask the VanillaBP team if you need a date for that.

## Source

Everything here follows
[Migrating from VanillaBP 1 to 2](https://github.com/vanillabp/adapter-platform-integration/wiki/Migrating-from-version-1).
That page decides, and it wins where this skill is out of date. Building or extending an
application is a different job with its own skill, `vanillabp`.
