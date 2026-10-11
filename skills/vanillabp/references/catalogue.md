# The blueprint catalogue

A blueprint is a small application you can run. It shows one aspect of building business
process software with VanillaBP. They all use the same package structure, the same placeholders
and the same test setup. So if you understand one, you understand all of them. To combine two,
you apply two deltas.

Blueprints are developed in one monorepo. They are delivered as one repository per blueprint
and platform, at `https://github.com/vanillabp-blueprints/<blueprint-id>-<platform>`, where
`<platform>` is `springboot` or `quarkus`. Those delivered repositories are read-only mirrors.
So issues and pull requests belong in
[the monorepo](https://github.com/vanillabp-blueprints/blueprints).

## Always prefer the live index

```
https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/blueprints.yaml
```

That file is the single source of truth. For each blueprint it carries the BPMN element types
it covers (`covers.bpmn`), the SPI members it uses (`covers.spi`), which blueprints it combines
with (`composes_with`), the blueprint it builds on (`base`), and the documentation it links
(`docs`). Per platform it carries the status, the repository and the URL of its `AGENTS.md`.

The tables below are a copy taken on 2026-08-22, for the case that you cannot reach the
network. Say so when you work from the copy, because a blueprint added since then is missing
here.

## From BPMN element to blueprint

Parse the model, collect the element types, then look them up. Several rows may apply to one
model, and that is normal. Each is a delta on top of `module-single`.

| BPMN element | Blueprint |
|---|---|
| `bpmn:ServiceTask` | `bpmn-service-task` |
| `bpmn:UserTask` | `bpmn-user-task` |
| `bpmn:SendTask` | `bpmn-async-task`, and `module-interaction` when the receiver is another workflow module |
| `bpmn:CallActivity` | `bpmn-call-activity-decomposition` |
| `bpmn:SubProcess` with `bpmn:MultiInstanceLoopCharacteristics` | `bpmn-multi-instance-subprocess` |
| `bpmn:MultiInstanceLoopCharacteristics` on a task | `bpmn-multi-instance-task` |
| `bpmn:ExclusiveGateway`, conditional `bpmn:SequenceFlow` | `bpmn-gateways` |
| `bpmn:ParallelGateway` | `persistence-parallel-branches` |
| `bpmn:StartEvent` with `bpmn:MessageEventDefinition` | `bpmn-message-start` |
| `bpmn:StartEvent` with `bpmn:TimerEventDefinition` or `bpmn:SignalEventDefinition` | `bpmn-bpms-initiated-start` |
| `bpmn:IntermediateCatchEvent` with `bpmn:MessageEventDefinition` | `bpmn-message-correlation` |
| `bpmn:IntermediateThrowEvent` | `module-interaction` |
| `bpmn:IntermediateCatchEvent` with `bpmn:TimerEventDefinition` | `bpmn-timer` |
| `bpmn:SignalEventDefinition` | `bpmn-signals` |
| `bpmn:BoundaryEvent` | `bpmn-boundary-events`, with a timer `bpmn-timer` |
| `bpmn:ErrorEventDefinition`, `bpmn:EscalationEventDefinition` | `bpmn-error-escalation` |
| `bpmn:EndEvent` the application has to learn about | `bpmn-workflow-ended` |

A parallel gateway, a non-interrupting boundary event and a multi-instance element each add a
token to the workflow. So `persistence-parallel-branches` matters for each of them too. Read it
before you let two branches write one aggregate.

## Workflow module structure and runtime

| Blueprint | What it shows | Platforms |
|---|---|---|
| `module-single` | Application plus one workflow module, the entry point | both |
| `module-multi` | Several workflow modules in one application | both |
| `module-standalone` | The application is the workflow module | both |
| `module-interaction` | Interaction between workflow modules | both |
| `module-packaging` | Assembling a runtime from workflow modules, and shipping it | both |
| `module-bpms-migration` | Migrating running workflows to another BPMS | both |

## Persistence of workflow aggregates

| Blueprint | What it shows | Platforms |
|---|---|---|
| `persistence-mongodb` | Workflow aggregates in MongoDB | both |
| `persistence-custom` | A persistence of your own | both |
| `persistence-parallel-branches` | Two branches writing one aggregate | both |
| `persistence-liquibase` | The application owns its database schema | both |
| `persistence-flyway` | The same with Flyway | both |
| `persistence-active-record` | Aggregates without a repository | Quarkus only, Spring Boot has no active record idiom for entities |

## BPMN scenarios

| Blueprint | What it shows | Platforms |
|---|---|---|
| `bpmn-service-task` | Service tasks | both |
| `bpmn-user-task` | User tasks | both |
| `bpmn-async-task` | Asynchronous tasks | both |
| `bpmn-message-correlation` | Messages for running workflows | both |
| `bpmn-message-start` | Starting a workflow by message | both |
| `bpmn-bpms-initiated-start` | Workflows the BPMS starts on its own | both |
| `bpmn-timer` | Timers | both |
| `bpmn-signals` | Signals | both |
| `bpmn-workflow-ended` | Learning that a workflow ended | both |
| `bpmn-boundary-events` | Boundary events | both |
| `bpmn-error-escalation` | BPMN errors and escalations | both |
| `bpmn-gateways` | Gateways and conditional sequence flows | both |
| `bpmn-call-activity-decomposition` | Call activities to reduce complexity | both |
| `bpmn-multi-instance-task` | Multi-instance tasks | both |
| `bpmn-multi-instance-subprocess` | Multi-instance subprocesses | both |
| `bpmn-versioning` | Versioning BPMN processes | both |
| `bpmn-aggregate-decoupling` | Decoupling BPMN from the data model | both |
| `bpmn-history-and-diagram` | Showing BPMN and execution history | both |

## Showcase

| Blueprint | What it shows | Platforms |
|---|---|---|
| `showcase-standalone` | A complete application | planned for both |

## Choosing the BPMS

A Maven profile decides which engine a blueprint runs on, never a code change: `-Pcamunda7`
(the default), `-Pcamunda8` or `-Pprocess-engine-api`. Keep that wiring when you copy a
blueprint. [project-structure.md](project-structure.md) describes it in the section about
configuration.

Blueprints are built and tested against Camunda 7 and Camunda 8. The Process-Engine-API adapter
is developed against a mock first and has no engine behind it. So no blueprint runs a workflow
on it yet.
