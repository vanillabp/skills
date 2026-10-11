# How a VanillaBP application is laid out

This repeats the structure every blueprint follows. The version which decides lives at
`https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`, and it wins
where the two disagree.

## Placeholders

Every blueprint uses the same four placeholders. Replace all of them consistently. Most startup
errors in generated applications come from replacing one and forgetting another.

| Placeholder | Meaning | Example |
|---|---|---|
| `blueprint.workflowmodule` | base package | `com.acme.orders` |
| `loanapproval` | use case identifier as a Java package | `shipment` |
| `loan-approval` | use case identifier in kebab case: workflow module ID, resource directory, REST path | `shipment` |
| `loan_approval` | BPMN process ID | `shipment` |

The BPMN process ID in the model has to match the one used in the code. The resource directory
has to match the workflow module ID.

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

An application built from several blueprints stays readable only if all of them agree on this
layout. So keep it.

That is the layout of one use case of medium size. A large process is split by section, and a
user task gets a package of its own. Both are described under
[rule 8](rules.md#8-one-primary-bpmn-process-per-workflow-aggregate). The classes keep their
names and their roles in every one of those packages.

## One class per direction

Talking to a BPMS (business process management system) goes in both directions, and the two are
different things in the architecture:

```
ApiController ─────────────────┐
                               ├──→ Service ──→ Workflow ──→ ProcessService     outgoing
BPMS ──→ WorkflowTaskHandler ──┘                                         incoming
```

`Workflow` is the outgoing half. `ProcessService` is injected here and nowhere else. `Service`
calls in and names what happened in business terms. This class translates that into what the
process needs.

`WorkflowTaskHandler` is the incoming half. It carries `@WorkflowService` and every
`@WorkflowTask` method, and it calls `Service`. It is a driving adapter, the same kind of thing
as `ApiController`. That the caller is a BPMS and not a browser changes nothing.

```java
// Service.java - what happened
public void submitRiskAssessment(final String id, final boolean acceptable) {
  final var loanApproval = loanApprovals.findById(id).orElseThrow();
  loanApproval.setRiskAcceptable(acceptable);
  workflow.riskAssessmentSubmitted(loanApproval);
}

// Workflow.java - what it means for the process
public void riskAssessmentSubmitted(final Aggregate loanApproval) {
  bpms.correlateMessage(loanApproval, "RiskAssessed");
}

// WorkflowTaskHandler.java - what the process wants from the application
@WorkflowTask
public void assessRisk(final Aggregate loanRequest, @TaskId final String taskId) {
  loanApproval.riskAssessmentRequested(loanRequest, taskId);
}
```

Name the methods of `Workflow` after the business event (`riskAssessmentSubmitted`), never
after the BPMN element (`correlateRiskAssessedMessage`). The model may be remodelled. A message
may become a timer, and a task a call activity, and the business code does not notice. That is
the whole point.

Name an injected bean after the role it plays in the class, not after its type. The `Service`
of a loan approval is `loanApproval`, `ProcessService` is `bpms`, `Workflow` is `workflow`, and
a repository is named after what it stores, `loanApprovals`. A call then reads like a sentence
about the business case. Where a parameter would hide the field, give it a name from the
domain, as `loanRequest` above.

Do not merge the two classes. If you put both directions into one class, it depends on
`Service` while `Service` depends on it. Spring Boot rejects such a circular bean reference at
startup, unless you work around it with `@Lazy`. An interface which `Service` implements does
not help either, because the cycle is between beans, not between types. Splitting by direction
removes the cycle instead of hiding it.

Keep both classes even where the translation is a single line. The seam costs nothing while a
process is simple. It keeps the business code readable once the process is not simple any more.

If you want to unit-test the business service without a BPMS, let `Workflow` implement an
interface which the application owns, and inject that. Do it on the outgoing side only. The
incoming adapter may depend on the application directly, exactly as `ApiController` does.

## A `@WorkflowTask` method contains no business logic

It translates and does nothing else. The BPMS delivers the aggregate, `@TaskId`, `@TaskEvent`
and the multi-instance element with its index. The method turns that into a call to `Service`.
With a single service task that leaves one line. That is honest, because there is nothing to
translate. In a multi-instance task or a user task it becomes real work: pick the element this
call is about, keep the task ID, react when the task was canceled. That work belongs there, not
in the business code.

The work behind a task is business code and lives in `Service`. The handler may call it,
because the directions are split. Logic about the business object itself may sit on the
aggregate, which is a normal entity. But that is a matter of taste, not a rule. The rule is
that nothing computes inside a `@WorkflowTask` method.

## Two namespaces per workflow module

Workflow modules are not isolated by classloaders. They end up in one runtime, on one
classpath. So two modules have to differ in two namespaces. Both are derived from the same
identifier.

| | Rule | Example |
|---|---|---|
| Classes | a unique Java package for the whole module | `com.acme.orders.shipment` |
| Resources | every resource in one subdirectory named after the workflow module ID | `src/main/resources/shipment/…` |

Every resource means every one: BPMN models in `<module-id>/processes/<adapter-id>/`, the
module's configuration file `<module-id>/<module-id>.yaml`, templates, documents, schemas. A
resource at the classpath root works fine until a second workflow module ships a file of the
same name. Then one of them wins, without a word.

The single exception is the marker file `META-INF/workflow-module`, which contains the module
ID. It has to sit at that exact path, because that is how the module is recognised at all.

## Configuration per BPMS goes into a profile file

`application.yaml` of the application module holds what every engine needs. Everything which
belongs to one engine goes into the profile file of that engine, `application-camunda7.yaml`
and `application-camunda8.yaml`. Where a test needs it, it also goes into the workflow module's
test resources. Never mix the two. A reader has to be able to see what changes when the engine
changes. And an application which will soon migrate from one BPMS to another runs with both
profiles side by side.

The BPMS is named once, on the Maven command line. Every BPMS profile of a blueprint sets the
property `bpms`. The build filters it into `application.yaml` as `spring.profiles.active` or
`quarkus.config.profile.parent`. Surefire and failsafe pass it to the tests. Keep that wiring
when you copy a blueprint. Otherwise the engine has to be named twice, and the two will drift
apart.

If you name an adapter id whose adapter is not on the classpath, VanillaBP refuses to start,
because that is a configuration error. So a profile file never applies to the wrong engine.

## What an application needs to declare

A workflow module depends on `io.vanillabp:vanillabp-spring-boot-support` or
`io.vanillabp:vanillabp-quarkus-support`. They bring the business SPI (service provider
interface) along, and they expose no BPMS API, on purpose. The adapter dependency belongs in
the application module. On Quarkus both VanillaBP and the adapter are Quarkus extensions, so
both are explicit dependencies there.

Adapter configuration lives at `vanillabp.adapters.<id>.*`, where the id names one BPMS
instance. An application with one adapter dependency and one workflow module needs no
`vanillabp.*` property at all. The classpath is the configuration, and everything else is
derived from it.
