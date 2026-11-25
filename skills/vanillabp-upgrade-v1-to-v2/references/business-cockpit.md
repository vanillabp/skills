# The Business Cockpit part

This applies to an application which reports to the
[VanillaBP Business Cockpit](https://github.com/vanillabp/business-cockpit). It is part of the
same upgrade, not a move of its own. The version 1 cockpit adapters ran only next to VanillaBP
1's `spring-boot-support`, so they go with it. Business Cockpit 1.0 brings their successors.

The Java code stays. `@UserTaskDetailsProvider`, `@WorkflowDetailsProvider`, `@DetailsEvent`,
the prefilled details objects and `BusinessCockpitService` keep their API. The keys stay too, in
the places version 1 read them. The dependency changes, a few keys are gone, two keys are new or
stricter, and a report is now sent after your commit.

A Business Cockpit of your own, meaning the cockpit application itself, is a different job. The
skill `business-cockpit-upgrade-to-blocking` covers it. Here it is only about an application
which reports to a cockpit.

## Does the application report to a cockpit?

| Look for | Command | Meaning |
|---|---|---|
| Version 1 cockpit adapter | `grep -rn "io.vanillabp.businesscockpit" -A1 --include=pom.xml . \| grep "spring-boot-adapter\|adapters-commons\|<artifactId>commons<"` | `camunda7-spring-boot-adapter` or `camunda8-spring-boot-adapter` of the group `io.vanillabp.businesscockpit`, with `adapters-commons` below them. The grep for `-spring-boot-adapter` in [detection.md](detection.md) hits these too. Their group tells them apart from the BPMS adapter |
| Cockpit keys | `grep -rn "cockpit" --include="*.yaml" --include="*.yml" --include="*.properties" .` | `vanillabp.cockpit.*` for the application, a `cockpit` section below a workflow module, a workflow or a user task |
| Details providers | `grep -rln "@UserTaskDetailsProvider\|@WorkflowDetailsProvider" --include="*.java" .` | they stay. Read them for the initiator and the `version` attribute below |

If none of these hits, skip this file.

Then add the section
[Up to 1.0.0](https://github.com/vanillabp/business-cockpit/blob/main/UPGRADE.md#up-to-100) of
the cockpit's `UPGRADE.md` to the checklist of step 0. Many of its entries are about the cockpit
application. Mark those "does not apply, because this application only reports". The section
[From 0.4.x and earlier](https://github.com/vanillabp/business-cockpit/blob/main/UPGRADE.md#from-04x-and-earlier)
is only for a cockpit of your own. Skip it.

## What else to grep for

| Look for | Command | Meaning |
|---|---|---|
| `commons` imports | `grep -rn "io.vanillabp.cockpit.commons" --include="*.java" .` | the version 1 adapter brought the artifact `commons` along, and it is gone. Such an import does not compile. The `UPGRADE.md` entry [`commons` is gone](https://github.com/vanillabp/business-cockpit/blob/main/UPGRADE.md#commons-is-gone-support-and-the-library-module-take-its-classes) has the new package of each class |
| Group hierarchy bean | `grep -rn "group-hierarchy-bean-name" .` | the key ends the start now, see below |
| Template directory | `grep -rn "template-loader-path" .` | the value needs `classpath:` or `file:` in front of it |
| Initiator | `grep -rn "initiator-source\|setInitiator" .` | `initiator-source` is new and has no default |
| Own transport | `grep -rn "UserTaskPublishing\|WorkflowPublishing\|WorkflowModulePublishing" --include="*.java" .` | publishing beans of your own. They become one `BusinessCockpitTransport` |
| Container annotation | `grep -rn "WorkflowDetailsProviders" --include="*.java" .` | no longer published. Delete the import. `UserTaskDetailsProviders` stays |
| Version attribute | `grep -rn "@UserTaskDetailsProvider\|@WorkflowDetailsProvider" -A3 --include="*.java" . \| grep version` | read now. Version 1 ignored it |
| Workflow provider API | `grep -rn "workflow-provider-api" --include=pom.xml .` | both artifacts are gone. Remove them from the POM |
| Kafka | `grep -rn "spring.kafka\|kafka-exporter" .` | the adapter builds its own producer now, see the keys below |

## The dependency

Remove the version 1 cockpit adapter and `adapters-commons`. Add the cockpit adapter for your
BPMS next to the adapter of the BPMS itself. All of them use the group
`io.vanillabp.businesscockpit`:

| BPMS | Spring Boot | Quarkus | Documentation |
|---|---|---|---|
| Camunda 7 | `businesscockpit-camunda7-adapter-spring-boot` | `businesscockpit-camunda7-adapter-quarkus` | [wiki](https://github.com/vanillabp/business-cockpit-camunda7-adapter/wiki) |
| Camunda 8 | `businesscockpit-camunda8-adapter-spring-boot` | `businesscockpit-camunda8-adapter-quarkus` | [wiki](https://github.com/vanillabp/business-cockpit-camunda8-adapter/wiki) |
| Process-Engine-API | `businesscockpit-process-engine-api-adapter-spring-boot` | `businesscockpit-process-engine-api-adapter-quarkus` | [wiki](https://github.com/vanillabp/business-cockpit-process-engine-api-adapter/wiki) |

The Camunda 8 artifact is released once per Camunda 8 minor version, like the BPMS adapter. Its
version ends in the same cluster line, for example `-8.9`. The Process-Engine-API adapter has no
version 1 predecessor. On Quarkus the part which is the same for every BPMS is an extension of
its own. Add it next to the BPMS extension, as the wiki of the adapter shows.

On Spring Boot, import the cockpit's BOM in `dependencyManagement`. It manages the versions of the
cockpit's own artifacts, such as `business-cockpit-spring-boot-support`, and imports
`io.vanillabp:vanillabp-bom` and Spring Boot's BOM as well. So the cockpit artifacts in the POM
lose their versions, and an import of `vanillabp-bom` next to it can go:

```xml
<dependency>
  <groupId>io.vanillabp.businesscockpit</groupId>
  <artifactId>business-cockpit-bom</artifactId>
  <version>1.0.0</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

The cockpit adapters are released on their own, in their own repositories, so they keep their
version. Do not import the BOM on Quarkus: it brings Spring Boot's versions, and those would win
over the ones Quarkus brings. There, name each version yourself.

A module which imported classes from `commons` depends on
`io.vanillabp.businesscockpit:business-cockpit-spring-boot-support` instead, or on
`business-cockpit-quarkus-support` on Quarkus. `support` brings the cockpit's SPI along. A
workflow module which is a library of its own takes the cockpit adapter in test scope only. The
application which runs it brings the adapter.

A report is now an outbox entry. So the application needs an outbox store, configured by
`vanillabp.outbox.*`. See [configuration.md](configuration.md#a-new-section-the-outbox).

## The keys

Every key which is not named here stays as it was. These nine are gone. A leftover one ends the
start, and the message says what became of it:

| Version 1 key | What to do |
|---|---|
| `rest.log` | set the logger `io.vanillabp.cockpit.bpms.api.v1_1.BpmsApi` to `DEBUG` |
| `rest.additional-get-parameters.<name>` | delete it. Every report is a POST |
| `rest.retry.*` | delete it. A failed report waits in the outbox, configured by `vanillabp.outbox.*` |
| `kafka.group-id-suffix` | delete it. The adapter only produces |
| `jwt.hmacSHA256-base64`, `jwt.cookie.*` | delete them. Version 1 never read them |
| `group-hierarchy-bean-name` | see the next section |
| `workerId` | delete it. The adapter takes the host name |
| `spring.application.name` | only the Camunda 8 exporter path read it, and that path is gone |
| `spring.kafka.consumer.*`, `camunda.zeebe.kafka-exporter.topic-name` | delete them. The adapter learns the same from the listeners it puts into the models |

Two keys move. The adapter used to take `spring.kafka.bootstrap-servers` and
`spring.kafka.producer.*`. Now it reads `vanillabp.cockpit.kafka.bootstrap-servers` and
`vanillabp.cockpit.kafka.properties.*`.

Two keys need a value now.

`initiator-source` has no default, so an application without it does not start. Ask the user
which value applies. `by-application` means the details providers set the initiator. Then a
report which names nobody fails, and the work of the engine fails with it. A provider uses
`Initiator.SYSTEM` for one event no user caused. `system` means the module knows no action a
user causes. A workflow module or a single workflow may set a value of its own. Existing cases
keep what they have, because nobody can tell later who started them.

```yaml
vanillabp:
  cockpit:
    initiator-source: by-application
```

`template-loader-path` needs `classpath:` or `file:` in front of its value. Version 1 read
`/opt/my-application/templates` as a directory of the file system, so that becomes
`file:/opt/my-application/templates`. The message repeats your value with both prefixes.

A workflow module now finds its templates in `<workflow-module-id>/business-cockpit-templates`
on the classpath without any key. As long as `vanillabp.cockpit.template-loader-path` is set,
nothing changes for modules which name no directory of their own. One case changes without a
key: a module with Freemarker on its classpath, no application-wide key, and a directory of that
name which held something else. Its templates are rendered now.

VanillaBP declares nothing below `vanillabp.cockpit`. The cockpit adapter reads that part
itself, and it now refuses a key it does not read, on Spring Boot too. The message names the key
it thinks you meant. That also catches a user task written below `tasks.<task>` instead of
`user-tasks.<task>`.

### `group-hierarchy-bean-name` becomes a bean

Write a fixed hierarchy below `vanillabp.workflow-modules.<id>.cockpit.group-hierarchy`. If the
hierarchy changes while the application runs, keep the bean and remove the key:

- import `io.vanillabp.spi.cockpit.workflowmodules.WorkflowModuleGroupHierarchy`,
- add `getWorkflowModuleId()`. The adapter finds the bean by its type and this id, not by its
  name,
- after a change, publish a `WorkflowModuleGroupHierarchyChangedEvent` with the module id
  through the `ApplicationEventPublisher`,
- ask the bean on every request. The adapter keeps no copy.

A module with the bean and `group-hierarchy` as well does not start. Neither does a bean for a
module which reports nothing to the cockpit.

### A module section is complete or empty

A workflow module which sets some of its cockpit keys and leaves others out ends the start. The
message names every gap in the application in one boot. A module which sets nothing at all is a
module which does not want to report. The application starts, and one INFO line per such module
names the key which would let it report. The cockpit never hears of that module. So read the
INFO lines of the first start. A module which reported in version 1 and stays quiet now is easy
to miss.

## Code

The details providers stay. Three things change around them:

- the `commons` imports, see the table above,
- `WorkflowDetailsProviders` is gone. Delete the import,
- publishing beans of your own. Version 1 let a `UserTaskPublishing`, `WorkflowPublishing` or
  `WorkflowModulePublishing` bean replace REST or Kafka. Version 2 has one bean type,
  `BusinessCockpitTransport`, with one method per kind of report. A bean which served only one
  kind wraps the shipped transport, as
  [Connecting a workflow module](https://github.com/vanillabp/business-cockpit/wiki/Connecting-a-workflow-module#rest-kafka-or-a-way-of-your-own)
  shows. Such an application configures neither `vanillabp.cockpit.rest.base-url` nor
  `vanillabp.cockpit.kafka.bootstrap-servers`.

## Behaviors that differ

Go through these with the user. They change what people see in the cockpit. Most need no edit.
Two of them may need a change to a model or a provider.

A report is sent after your commit. It is written in the transaction which caused it and sent
once that transaction committed. So a details provider runs at dispatch time. If it changes an
aggregate, the change is saved in that later transaction. In return, a report survives a cockpit
server which is down, and a burst of changes to one task arrives as one report.

A task is read when the report is sent, not when the event happened. That goes for the assignee,
the candidates, the dates and the values a `@TaskParam` is bound from.

Every user task shows up. Version 1 dropped a task which no `@UserTaskDetailsProvider` matched,
and said nothing. Now every user task is reported. Expect new tasks in the cockpit, and give them
a provider or a template where the BPMN name makes a poor title.

Who sees a task is set once. The report which creates the task sets the assignee, the candidates
and the excluded users. Later reports do not change them. A module which handed a task to other
people by reporting it again has to end the task and enter it again, for example with a boundary
event.

The `version` attribute picks the method now, on `@UserTaskDetailsProvider` and also on
`@WorkflowDetailsProvider`. Version 1 ignored it, so a method with a `version` served every model
version. Check each hit of the grep. A method without the attribute still serves every version.
[Which versions of your model a method serves](https://github.com/vanillabp/business-cockpit/wiki/Reporting-workflows-and-user-tasks#which-versions-of-your-model-a-method-serves)
has the details.

An application which extends the dev shell simulator now gets its defaults, among them
`spring.jpa.hibernate.ddl-auto=update` and `spring.jpa.open-in-view=false`. If the application
relies on open-in-view, it sets it to `true` itself. The upgrade notes of the cockpit,
[UPGRADE.md](https://github.com/vanillabp/business-cockpit/blob/main/UPGRADE.md), have the details.

The start checks everything. Version 1 found a broken setting at the first event, or never. Now
the log of the first start names every gap.

### Camunda 8

The cockpit's three task listeners of a user task stay the same. But the start of a workflow is
now reported from an execution listener on every start event. So the first deployment after the
upgrade creates a new process version. Running workflows stay on their version and keep
reporting their user tasks.

The version 1 adapter created a table `CAMUNDA8_BC_PROCESS_INSTANCES` and never wrote a row into
it. Nothing removes it. It can be dropped by hand without losing data.

### Camunda 7

The engine has to write history. Every history level from `activity` up works, including the
default `audit`. An engine whose history is switched off ends the start.

## What ends the first start

Each of these stops the application, and the message names the way out:

- a key of version 1 which is gone,
- a key the adapter does not read,
- no `initiator-source`,
- a `template-loader-path` without `classpath:` or `file:`,
- `group-hierarchy-bean-name`, or a hierarchy bean next to `group-hierarchy`, or a bean for a
  module which does not report,
- a workflow module whose cockpit section is incomplete,
- on Camunda 7, an engine without history.

The `commons` imports fail before that, in the compiler.

## Sources

[Migrating from version 1](https://github.com/vanillabp/business-cockpit/wiki/Migrating-from-version-1)
in the cockpit's wiki, section 6 of
[Migrating from VanillaBP 1 to 2](https://github.com/vanillabp/adapter-platform-integration/wiki/Migrating-from-version-1#6-business-cockpit),
and the *Migrating from version 1* page of the wiki for your BPMS
([Camunda 7](https://github.com/vanillabp/business-cockpit-camunda7-adapter/wiki/Migrating-from-version-1),
[Camunda 8](https://github.com/vanillabp/business-cockpit-camunda8-adapter/wiki/Migrating-from-version-1)).
The keys of each adapter are on the *Configuration* page of the same wiki.
