# How a VanillaBP application is laid out

This restates the structure every blueprint follows. The authoritative version lives at
`https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md` and wins where
the two disagree.

## Placeholders

Every blueprint uses the same four placeholders. Replace all of them consistently, because most
startup errors in generated applications come from replacing one and forgetting another.

| Placeholder | Meaning | Example |
|---|---|---|
| `blueprint.workflowmodule` | base package | `com.acme.orders` |
| `loanapproval` | use case identifier as a Java package | `shipment` |
| `loan-approval` | use case identifier in kebab case: workflow module ID, resource directory, REST path | `shipment` |
| `loan_approval` | BPMN process ID | `shipment` |

The BPMN process ID in the model has to match the one used in the code, and the resource
directory has to match the workflow module ID.

## Reference structure

```
<base-package>.<usecase>
├── ApiController.java               <- driving adapter: HTTP calls in
├── Service.java                     <- business code, never touches VanillaBP
├── Workflow.java                    <- outgoing: the application tells the process
├── WorkflowTaskHandler.java         <- incoming: the process tells the application
├── config/<UseCase>Properties.java
└── model/
    ├── Aggregate.java
    └── AggregateRepository.java
```

An application assembled from several blueprints stays readable only if all of them agree on
this layout, so keep it.

## One class per direction

Talking to a BPMS happens in both directions and the two are different architectural things:

```
ApiController ──────────┐
                        ├──→ Service ──→ Workflow ──→ ProcessService     outgoing
BPMS ──→ WorkflowTaskHandler ──┘                                         incoming
```

`Workflow` is the outgoing half. `ProcessService` is injected here and nowhere else. `Service`
calls in, naming what happened in business terms, and this class translates that into what the
process needs.

`WorkflowTaskHandler` is the incoming half. It carries `@WorkflowService` and every
`@WorkflowTask` method, and it calls `Service`. It is a driving adapter, the same kind of thing
as `ApiController`: that the caller is a BPMS rather than a browser changes nothing.

```java
// Service.java - what happened
public void submitRiskAssessment(final String id, final boolean acceptable) {
  final var loanApproval = loanApprovals.findById(id).orElseThrow();
  loanApproval.setRiskAcceptable(acceptable);
  workflow.riskAssessmentSubmitted(loanApproval);
}

// Workflow.java - what it means for the process
public void riskAssessmentSubmitted(final Aggregate loanApproval) {
  processService.correlateMessage(loanApproval, "RiskAssessed");
}

// WorkflowTaskHandler.java - what the process wants from the application
@WorkflowTask
public void assessRisk(final Aggregate loanApproval, @TaskId final String taskId) {
  service.riskAssessmentRequested(loanApproval, taskId);
}
```

Name the methods of `Workflow` after the business event (`riskAssessmentSubmitted`) and never
after the BPMN element (`correlateRiskAssessedMessage`). The model may be remodelled, a message
may become a timer and a task a call activity, without the business code noticing, and that is
the whole point.

Do not merge the two classes. Putting both directions into one makes it depend on `Service`
while `Service` depends on it, a circular bean reference which Spring Boot rejects at startup
unless it is worked around with `@Lazy`. An interface implemented by `Service` does not help
either, because the cycle is between beans and not between types. Splitting by direction
removes it instead of hiding it.

Keep both classes even where the translation is a single line. The seam costs nothing while a
process is trivial and is what keeps the business code readable once it is not.

If the business service is to be unit-testable without a BPMS, let `Workflow` implement an
interface owned by the application and inject that. Do it on the outgoing side only. The
incoming adapter may depend on the application directly, exactly as `ApiController` does.

## A `@WorkflowTask` method contains no business logic

It translates and nothing else. What the BPMS delivers is the aggregate, `@TaskId`, `@TaskEvent`
and the multi-instance element with its index, and the method turns that into a call to
`Service`. With a single service task that leaves one line, which is honest, since there is nothing to
translate. In a multi-instance task or a user task it becomes real work: pick the element this
invocation is about, keep the task ID, react to the task having been canceled. That work belongs
there rather than in the business code.

The work behind a task is business code and lives in `Service`, which the handler may call
because the directions are split. Logic about the business object itself may sit on the
aggregate, which is a normal entity, but that is a matter of taste and not a rule. The rule is
that nothing computes inside a `@WorkflowTask` method.

## Two namespaces per workflow module

There is no classloader isolation between workflow modules. They end up in one runtime on one
classpath, so two modules have to differ in two namespaces, both derived from the same
identifier.

| | Rule | Example |
|---|---|---|
| Classes | a unique Java package for the whole module | `com.acme.orders.shipment` |
| Resources | every resource in one subdirectory named after the workflow module ID | `src/main/resources/shipment/…` |

Every resource means every one: BPMN models in `<module-id>/processes/<adapter-id>/`, the
module's configuration file `<module-id>/<module-id>.yaml`, templates, documents, schemas. A
resource placed at the classpath root works fine until a second workflow module ships a file of
the same name, and then one of them silently wins.

The single exception is the marker file `META-INF/workflow-module` containing the module ID. It
has to sit at that exact path, which is how the module is recognised at all.

## Configuration per BPMS goes into a profile file

`application.yaml` of the application module holds what every engine needs. Everything belonging
to one engine goes into the profile file of that engine, `application-camunda7.yaml` and
`application-camunda8.yaml`, and into the workflow module's test resources where a test needs
it. Never mix the two. A reader has to be able to see what changes when the engine changes, and
an application about to migrate from one BPMS to another runs with both profiles side by side.

The BPMS is named once, on the Maven command line. Every BPMS profile of a blueprint sets the
property `bpms`, the build filters it into `application.yaml` as `spring.profiles.active`
respectively `quarkus.config.profile.parent`, and surefire and failsafe pass it to the tests.
Keep that wiring when you copy a blueprint, otherwise the engine has to be named twice and the
two will drift apart.

Naming an adapter id whose adapter is not on the classpath is a configuration error VanillaBP
refuses to start with, so a profile file never applies to the wrong engine.

## What an application needs to declare

A workflow module depends on `io.vanillabp:vanillabp-spring-boot-support` respectively
`io.vanillabp:vanillabp-quarkus-support`, which bring the business SPI transitively and
deliberately expose no BPMS API. The adapter dependency belongs into the application module. On
Quarkus both VanillaBP and the adapter are Quarkus extensions, so both are explicit dependencies
there.

Adapter configuration lives at `vanillabp.adapters.<id>.*`, where the id names one BPMS instance.
An application with one adapter dependency and one workflow module needs no `vanillabp.*`
property at all, because the classpath is the configuration and everything else is derived.
