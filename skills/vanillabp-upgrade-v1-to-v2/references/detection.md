# Surveying a version 1 project

Run these from the project root before changing anything. Each row names what to look for and
what a hit means. Adapt the tool to what is available; the patterns matter, not the command.

## Runtime and dependencies

| Look for | Command | Meaning |
|---|---|---|
| VanillaBP version | `grep -rn "io.vanillabp\|org.camunda.community.vanillabp" --include=pom.xml --include="*.gradle*" .` | a `1.x` version confirms this is a version 1 project |
| Old adapter artifact | `grep -rn -- "-spring-boot-adapter" --include=pom.xml .` | renamed in version 2, see [dependencies.md](dependencies.md) |
| Java version | `grep -rn "maven.compiler\|java.version\|<release>" --include=pom.xml .` | has to become 21 |
| Platform version | `grep -rn "spring-boot-starter-parent\|quarkus.platform.version" --include=pom.xml .` | has to become Spring Boot 4.1 or Quarkus 3.37 |
| Camunda 8 cluster minor | not in the project: ask which cluster the application talks to, or read the gateway version off its topology | the Camunda 8 adapter is published once per cluster minor and the minor is part of its version, see [dependencies.md](dependencies.md) |

## Workflow modules

| Look for | Command | Meaning |
|---|---|---|
| Existing descriptors | `find . -path "*/META-INF/workflow-module"` | version 2 needs one per module, containing the module id |
| How the id is derived today | `grep -rn "spring.application.name" --include="*.yaml" --include="*.yml" --include="*.properties" .` | single-module applications took the id from here |
| Module declared by a bean | `grep -rn "WorkflowModuleProperties" --include="*.java" .` | the third version 1 way of naming a module. The type is gone in version 2, so the bean method has to go and the id moves into the descriptor |
| Module configuration files | `find . -name "*.yaml" -o -name "*.properties" \| grep -v application` | a module's own `<module-id>.yaml` also named the module |
| Resource directories | `find . -path "*/src/main/resources/*/processes/*" -type d` | the directory name is the module id and has to stay the same |
| Decision tables | `grep -rln --include="*.dmn" . ` | version 2 deploys a `.dmn` file at the module's resources location together with the BPMN files, which version 1 never did. Find out how the project deploys its decisions today, because after the upgrade two things write the same one. Same bytes on both sides means nothing happens; different versions means one of the two has to go. Under `name-clash-avoidance: use-prefix` also read every business rule task: its reference is rewritten with the decision ids of the module, so one pointing at a decision this module does not deploy breaks |

Write down the id of every module. Keeping them is not optional: they name BPMS tenants and
resource directories, and a changed id detaches the running workflows of that module from their
configuration.

## Configuration

| Look for | Command | Meaning |
|---|---|---|
| `default-adapter` | `grep -rn "default-adapter" .` | becomes `prioritized-adapters`, a list, at all three levels |
| Per-module adapter settings | `grep -rn "workflow-modules" -A5 . \| grep -n "adapters"` | move to `vanillabp.adapters.<id>.*` |
| Camunda 7 engine settings | `grep -rn "camunda.bpm\|camunda:" --include="*.yaml" --include="*.yml" --include="*.properties" .` | the Camunda Spring Boot starter is gone in version 2 |
| Camunda 8 client settings | `grep -rn "zeebe.client\|camunda.client" .` | replaced by `vanillabp.adapters.<id>.*` |
| Tenants | `grep -rn "use-tenants\|tenant-id" .` | `use-tenants` becomes `name-clash-avoidance`; `by-adapter` is the default and matches version 1, so only `use-tenants: false` needs a line |
| Listeners in the model | `grep -rn "taskListener\|executionListener" --include="*.bpmn" .` | a listener served by a `@WorkflowTask` method has to be allowed now, `vanillabp.adapters.<id>.allow-listeners`, and a model carrying one ends the boot until it is. Read what the key costs before setting it, because a listener is where the model stops being portable between BPMS. On Camunda 8 only a `zeebe:taskListener` of a `zeebe:userTask` was served in version 1, from 1.7.0 on |
| Camunda 8 user tasks | `grep -rn "formKey" src/main/resources` | a `formKey` on a user task is the construction VanillaBP 1 dropped in 1.7.0 and version 2 does not serve. The model has to change, and the user tasks open on it have to be finished BEFORE the upgrade |
| Task timeout | `grep -rn "task-timeout" .` | renamed to `job-timeout` |
| Hexadecimal task ids | `grep -rn "task-id-as-hex-string" .` | if this was ever on, the task ids the application STORED are hexadecimal and version 2 reads decimally. A data migration, not a setting |
| Async definitions | `grep -rn "use-bpmn-async-definitions" .` | not configurable any more |
| Connectors allowed | `grep -rn "allow-connectors" .` | the key is back, under the adapter: `vanillabp.adapters.<id>.allow-connectors`, at adapter, workflow-module and workflow level, default `false` as before. Two things changed. The most specific level now wins in both directions, where version 1's booleans could only switch it on, so a `false` below a `true` really switches it off. And a user task built from an element template stays wired, where version 1 passed it over, so it needs a `@WorkflowTask` method |
| Removed keys | `grep -rn "vanillabp.resilience" .` | not in version 2, tell the VanillaBP team if the project relies on it |
| Retry backoff | `grep -rn "retry-backoff" .` | unchanged as a property, `vanillabp.adapters.<id>.retry-backoff`, resolvable down to a single task - but it defaults to `PT10S` now where version 1 sent none |
| Retry backoff in the model | `grep -rn "retryBackoff" --include="*.bpmn" .` | a `retryBackoff` TASK HEADER per element is read in both versions, and version 2 no longer needs the `retries` attribute next to it. Nothing to do, unless the task is also configured: then the TASK-level property wins over the header, while every level above the task loses to it. So a value moved up into adapter or module configuration during the upgrade is overruled by a header the application meant to retire. A header which is no ISO-8601 duration warns once per element and leaves the configured value in force |
| Resources location | `grep -rn "resources-location" .` | still valid but optional, usually deletable |

Search every profile file and every test resource, not only `application.yaml`. Test
configuration is where an upgrade is most often left half done.

## Code

| Look for | Command | Meaning |
|---|---|---|
| Removed overloads | `grep -rn "correlateMessage(\|startWorkflowByMessage(" --include="*.java" .` | the variants taking a message object are gone, inspect each hit |
| `primary` | `grep -rn "@BpmnProcess" -A2 --include="*.java" . \| grep primary` | the attribute no longer exists |
| Aggregate repositories | `grep -rn "Repository<" --include="*.java" . \| grep -i aggregate` | version 1 required one per aggregate, and version 2 still uses it. On Spring Boot a missing one is not reported at startup, only at the first task delivery, as `No Spring Data repository defined for '<class>'!` |
| Transaction annotations | `grep -rn "@Transactional" --include="*.java" .` | see [code-changes.md](code-changes.md), and check superclasses, interfaces and custom annotations too |
| Dead annotation | `grep -rn "javax.transaction.Transactional" --include="*.java" .` | honored by neither Spring Framework 7 nor Quarkus 3, it does nothing |
| Own implementations | `grep -rn "implements ProcessService" --include="*.java" .` | `ProcessService` gained methods, all of them `default`, so test doubles keep compiling |
| Workflow services | `grep -rln "@WorkflowService" --include="*.java" .` | read every one of them. Two classes declaring a DIFFERENT `bpmnProcess` for one aggregate fail the boot in version 2, so the second process moves into the first class' `secondaryBpmnProcesses`. Several classes on the SAME `bpmnProcess` stay as they are |
| A workflow service which is no bean | the same files: does each class carry `@Service`, `@Component` or a `@Bean` method returning it? | version 2 no longer scans the classpath for the annotation. On Spring Boot it walks the bean definitions and keeps the classes carrying it, so a workflow service which is no bean gets no `ProcessService` any more. The application still starts, and the deployment warns per workflow module about the BPMN processes nothing claims, naming the class and the ways to make it a bean. On Quarkus the same class fails the build instead. A `@Bean` method has to declare the workflow service class as its RETURN type, because the discovery reads the type of the definition and never the instance. A class registered by one profile only is found while that profile is active |
| Combined workflow and business service | read the same files | a class holding both `@WorkflowTask` methods and the business methods the API calls. Deleting its `@Transactional` takes the transaction away from the business methods too, and no grep decides this reliably |
| Version attribute | `grep -rn "@WorkflowTask\|@WorkflowStartedByBpms\|@WorkflowEnded\|@BpmnProcess" -A2 --include="*.java" . \| grep "version"` | evaluated for real now, so overlapping ranges fail the boot. The attribute of `@BpmnProcess` counts too: it is the fallback of every method which names no range of its own, and nothing read it in version 1 either. A class which names a range and a BPMS which reports no version do not go together, because such a delivery reaches methods without a range only |
| Sync annotations | `grep -rn "@SyncWithBPMS\|@NoSyncWithBPMS" --include="*.java" .` | shipped for real now, and annotating one attribute decides what happens to all the others. Finding nothing is the case to act on: an aggregate which holds nothing back shares everything, and version 2 refuses to start such a workflow until the aggregate says what the models need or the workflow allows the full sync |
| `@TaskParam` types | `grep -rn "@TaskParam" -A1 --include="*.java" .` | read the declared type of every one of them against the value the model maps in. A number is converted only where the conversion keeps the value, so `int` or `long` against a value which can carry a fraction or exceed the type, and `Double` or `Float` against a whole number larger than the type holds exactly, end the task instead of arriving wrong. Widen the type or map a value which fits. Version 1 refused these pairs too, so a project coming straight from version 1 finds nothing here; one which ran on a VanillaBP 2 snapshot in between may |
| Date and Calendar attributes of an aggregate | `grep -rn "java.util.Date\|java.util.Calendar\|java.util.TimeZone" --include="*.java" .` | a `Date` attribute which is shared reaches the BPMS as `2026-09-16T19:55:30.123Z` instead of `Wed Sep 16 21:55:30 CEST 2026`, so ask whether a BPMN expression compares that text. A `Calendar` attribute which is shared stops the application from starting and has to become an `Instant`, or be marked `@NoSyncWithBPMS`. A `TimeZone` attribute used to end the sync point and works now |
| Aggregate attributes a BPMS-initiated start writes | `grep -rn "@WorkflowStartedByBpms" --include="*.java" .` | the process variables such a start carries are written into the aggregate through the same conversion, so check the type of every attribute a start event's model sets |

## BPMN models, Camunda 7 only

| Look for | Command | Meaning |
|---|---|---|
| Expressions | `grep -rno '\${[^}]*}' --include="*.bpmn" .` | version 2 resolves these from process variables instead of reading the aggregate live |
| Expressions reading past a dot | the same hits: every expression with a dot in it | such an expression reads a nested value, which Camunda 7 stores as an object variable in whatever serialization format the engine was told to use. It needs `vanillabp.adapters.<id>.serialization-format` and a dataformat plugin on the classpath, or the engine falls back to Java serialization. The migration fallback does not cover these at all, because it answers a top-level name only |

For each expression check that the attribute it reads is reachable through a getter. Two forms
were legal in version 1 and are not covered by the sharing rules: an attribute readable only as
a field, and an `isX()` method returning something other than `boolean`. Give those a `getX()`,
or an `isX()` returning `boolean`, and share it.

The Camunda 7 adapter helps with the rest. While the application starts it names the expressions
whose value the engine does not hold, with the element, the segment the path stops at and what the
engine does with the resulting null. Read that report, and keep reading the models as well: the
check reads declared types, so it says nothing where a path runs through a `Map`, an interface or a
collection without an element type, and it judges no method call, which leaves `${order.getTotal()}`
out although it stops working. An expression like that raises an incident, so the engine reports
it. The silent half is what the startup report exists for.

## What no grep will find

Say these out loud in the report, because they need a person to answer:

- A transaction annotation on a bean that a `@WorkflowTask` method calls. No startup check can
  see it, so the task fails at runtime instead.
- Whether the application relies on the tenant a workflow module was deployed into. It does if
  it has running workflows on Camunda 7.
- Guards inside handlers written against at-least-once delivery. They stay correct and are still
  needed for concurrent deliveries, so leave them alone.
- Service tasks modelled in front of every end event only to learn that a workflow ended.
  `@WorkflowEnded` replaces them, but the old model keeps working, so this is a simplification
  for later, not an upgrade step.
