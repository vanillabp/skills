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

## Workflow modules

| Look for | Command | Meaning |
|---|---|---|
| Existing descriptors | `find . -path "*/META-INF/workflow-module"` | version 2 needs one per module, containing the module id |
| How the id is derived today | `grep -rn "spring.application.name" --include="*.yaml" --include="*.yml" --include="*.properties" .` | single-module applications took the id from here |
| Module declared by a bean | `grep -rn "WorkflowModuleProperties" --include="*.java" .` | the third version 1 way of naming a module. The type is gone in version 2, so the bean method has to go and the id moves into the descriptor |
| Module configuration files | `find . -name "*.yaml" -o -name "*.properties" \| grep -v application` | a module's own `<module-id>.yaml` also named the module |
| Resource directories | `find . -path "*/src/main/resources/*/processes/*" -type d` | the directory name is the module id and has to stay the same |

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
| Camunda 8 user tasks | `grep -rn "formKey" src/main/resources` | a `formKey` on a user task is the construction VanillaBP 1 dropped in 1.7.0 and version 2 does not serve. The model has to change, and the user tasks open on it have to be finished BEFORE the upgrade |
| Task timeout | `grep -rn "task-timeout" .` | renamed to `job-timeout` |
| Hexadecimal task ids | `grep -rn "task-id-as-hex-string" .` | if this was ever on, the task ids the application STORED are hexadecimal and version 2 reads decimally. A data migration, not a setting |
| Async definitions | `grep -rn "use-bpmn-async-definitions" .` | not configurable any more |
| Removed keys | `grep -rn "allow-connectors\|vanillabp.resilience" .` | not in version 2, tell the VanillaBP team if the project relies on them |
| Retry backoff | `grep -rn "retry-backoff" .` | unchanged as a property, `vanillabp.adapters.<id>.retry-backoff`, resolvable down to a single task - but it defaults to `PT10S` now where version 1 sent none |
| Retry backoff in the model | `grep -rn "retryBackoff" --include="*.bpmn" .` | version 1 read a `retryBackoff` TASK HEADER per element, version 2 does not. Move the value into the configuration of that task, nothing reports the loss |
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
| Combined workflow and business service | read the same files | a class holding both `@WorkflowTask` methods and the business methods the API calls. Deleting its `@Transactional` takes the transaction away from the business methods too, and no grep decides this reliably |
| Version attribute | `grep -rn "@WorkflowTask" -A2 --include="*.java" . \| grep "version"` | evaluated for real now, so overlapping ranges fail the boot |
| Sync annotations | `grep -rn "@SyncWithBPMS\|@NoSyncWithBPMS" --include="*.java" .` | shipped for real now, and annotating one attribute decides what happens to all the others |

## BPMN models, Camunda 7 only

| Look for | Command | Meaning |
|---|---|---|
| Expressions | `grep -rno '\${[^}]*}' --include="*.bpmn" .` | version 2 resolves these from process variables instead of reading the aggregate live |

For each expression check that the attribute it reads is reachable through a getter. Two forms
were legal in version 1 and are not covered by the sharing rules: an attribute readable only as
a field, and an `isX()` method returning something other than `boolean`. Give those a `getX()`,
or an `isX()` returning `boolean`, and share it.

You do not have to find them all by hand. The Camunda 7 adapter lists every expression needing
attention while the application starts, and that list is the authoritative backlog.

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
