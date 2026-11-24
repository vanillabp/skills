---
name: vanillabp-upgrade-v1-to-v2
description: Upgrade an application from VanillaBP 1 to VanillaBP 2. Surveys a version 1 project for everything that has to change, then applies it. Covers Java 21 and Spring Boot 4 or Quarkus 3.37, the renamed adapter artifacts, the META-INF/workflow-module descriptor, default-adapter becoming prioritized-adapters, adapter settings moving to vanillabp.adapters, the removed ProcessService overloads, @BpmnProcess.primary, and the transaction annotations on workflow services. Also covers the first start refusing an aggregate which shares everything, workflow services being found because they are beans, the version attribute of @BpmnProcess taking effect, decision tables deployed with their workflow module, the Camunda 8 wait for its cluster, the Camunda 7 serialization format a nested value needs, and reporting to the Business Cockpit. Use whenever a project depends on VanillaBP 1.x, whenever someone asks about upgrading or migrating to VanillaBP 2, or whenever a build against VanillaBP 2 fails on those symbols.
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
- every hand-written `ProcessService` implementation, test doubles included,
- whether the application reports to the Business Cockpit: a version 1 cockpit adapter of the
  group `io.vanillabp.businesscockpit`, or keys below `vanillabp.cockpit`. If it does, work
  through [references/business-cockpit.md](references/business-cockpit.md) as well.

Show that list to the user, with the count per item. It is also the checklist you tick off.

Then add the upgrade notes of the repositories to that checklist. Each repository keeps an
`UPGRADE.md` which lists every change a version 1 application has to deal with. Read these files
on the `main` branch:

- always
  [spi-for-java](https://github.com/vanillabp/spi-for-java/blob/main/UPGRADE.md) and
  [adapter-platform-integration](https://github.com/vanillabp/adapter-platform-integration/blob/main/UPGRADE.md),
- on Camunda 7, the file of the
  [Camunda 7 adapter](https://github.com/vanillabp/camunda7-adapter/blob/main/UPGRADE.md),
- on Camunda 8, the file of the
  [Camunda 8 adapter](https://github.com/vanillabp/camunda8-adapter/blob/main/UPGRADE.md),
- on CIB seven, the file of the
  [CIB seven adapter](https://github.com/vanillabp/cibseven-adapter/blob/main/UPGRADE.md). VanillaBP 1
  had no adapter for CIB seven. So that file sends you on to the Camunda 7 file and says what
  differs. Read both.
- on Operaton, the Camunda 7 file. VanillaBP 1 had no adapter for Operaton either, and the
  Operaton adapter has no `UPGRADE.md`. Its README says how it differs from the Camunda 7 adapter.
- if the application reports to the Business Cockpit, the section
  [Up to 1.0.0](https://github.com/vanillabp/business-cockpit/blob/main/UPGRADE.md#up-to-100) of
  the cockpit's file. Many of its entries are about the cockpit application itself. They do not
  apply to an application which only reports.

Every section of a file is one item. That means each heading below `## 2.0`, down to the
smallest one, and in spi-for-java also the `1.1.0` list, because a version 1 application skips
that line too. Tick each item off, or mark it "does not apply, because ...". Several items only
end the start of the application. You want to find those here, not at the first boot.

### Step 1: lift the runtime

Java 21, Spring Boot 4.1 or Quarkus 3.37. Aim at Spring Boot 4.1, also when a newer Spring Boot
is out. Spring Boot 4.2 needs the dependencies of VanillaBP 2, so it comes after this upgrade.

If the application is not on Spring Boot 4.1 yet, it first moves there while it is still on
VanillaBP 1. Upgrade Spring Boot 4.1 together with the last releases of version 1, which support
it: `io.vanillabp:spring-boot-support` 1.4.0, the Camunda 7 adapter 1.6.0, the Camunda 8 adapter
1.10.1 and, if the application reports to the Business Cockpit, its adapter 0.4.3.
[references/dependencies.md](references/dependencies.md) has the coordinates. Get the application
running again and commit, then move to VanillaBP 2. That keeps the changes of Spring Boot 4 and
those of VanillaBP 2 apart.

Spring Boot 4 split `spring-boot-autoconfigure` into
modules. So some imports of your own code move, and the test slices need explicit dependencies.
That is Spring's change, not VanillaBP's. Follow Spring Boot's own migration guide for it.

### Step 2: swap the dependencies

The adapter artifacts were renamed, and every adapter now has a Quarkus artifact too. The table
is in [references/dependencies.md](references/dependencies.md). On Camunda 8 the version also
names the minor version of the cluster you talk to. So decide which one that is before you
write the coordinate. Then build. The compiler points straight at the three removed overloads
and at `@BpmnProcess.primary`. Persistence needs nothing here. What version 1 could do, JPA and
MongoDB through Spring Data, version 2 does out of the box.

An application which reports to the Business Cockpit swaps its cockpit adapter in the same step.
[references/business-cockpit.md](references/business-cockpit.md) has the artifacts. Imports from
the cockpit's `commons` stop compiling here.

### Step 3: declare the workflow modules, then move the configuration

Version 2 wants the module id stated explicitly in a `META-INF/workflow-module` descriptor.
Version 1 derived it from `spring.application.name`, from the name of the module's
configuration file, or from a static bean method which returns a `WorkflowModuleProperties`.
That type is gone. So find every one of the three and replace it with the descriptor. Keep the
ids you had. They name BPMS tenants and resource directories. If you change one, the running
workflows of that module lose their configuration.

Then move the configuration key by key, following
[references/configuration.md](references/configuration.md). Start the application afterwards
and follow the startup messages. They name every missing key. Most cockpit keys stay where they
are. [references/business-cockpit.md](references/business-cockpit.md) lists the ones which are
gone, and says when a module's cockpit section ends the start. Ask the user for the value of
`initiator-source`, because it has no default.

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

If the application reports to the Business Cockpit, check that every workflow module which
reported before reports again. A module without a cockpit section starts quietly and reports
nothing. The first start logs one INFO line for each such module.

Then read [references/behavior-changes.md](references/behavior-changes.md) with the user. Those
are the things which changed although your code did not. Several of them need a decision, not
an edit.

Before you call the upgrade done, go through the `UPGRADE.md` items from step 0 once more. Every
one of them should be ticked off or carry its reason.

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
That page decides, and it wins where this skill is out of date.

The `UPGRADE.md` files from step 0 are the list to check against, so that nothing is left out.
They are written for contributors. So they are shorter about the reasons, and where one of them
and the wiki page disagree, the wiki page wins. Building or extending an
application is a different job with its own skill, `vanillabp`.

The Business Cockpit part follows the cockpit's own
[Migrating from version 1](https://github.com/vanillabp/business-cockpit/wiki/Migrating-from-version-1)
page. Upgrading a Business Cockpit of your own, the cockpit application itself, is another job
with its own skill, `business-cockpit-upgrade-to-blocking`.
