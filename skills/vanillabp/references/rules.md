# Rules that hold regardless of blueprint

Each of these is a way a VanillaBP application quietly stops being one. They are restated from
`https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`, which wins where
the two disagree.

## 1. No process variables

All state lives in the workflow aggregate. Sequence flow conditions call methods on the
aggregate instead of reading variables.

## 2. No BPMS API in application code

No `RuntimeService`, no `ZeebeClient`, no `CamundaClient`, no other engine class. The only
compile-time dependency towards the BPMS is the VanillaBP SPI. Reaching for an engine API means
the hexagonal architecture is broken and the application can no longer change its BPMS.

## 3. No BPMS dependency in a workflow module

A pure workflow module depends on `io.vanillabp:vanillabp-spring-boot-support` respectively
`io.vanillabp:vanillabp-quarkus-support` and never on an adapter. Those modules deliberately do
not expose BPMS APIs. The adapter dependency belongs into the application module.

## 4. No business state in the BPMN

The model expresses the flow, not the data. Nothing an auditor would want to read belongs
into the diagram.

## 5. One aggregate per workflow

Do not share an aggregate between two processes unless the blueprint
`bpmn-call-activity-decomposition` explicitly shows it, and do not persist two business objects
into one aggregate.

## 6. No direct message exchange between workflow modules

A process must never send a message to a process of another workflow module, because that
couples them. Use the way `module-interaction` shows: an outgoing message becomes an internal
API call, and the receiving module delivers a message through its own API.

## 7. Do not let two branches of one workflow write the aggregate carelessly

A parallel gateway, a non-interrupting boundary event and a multi-instance task each add a token,
so a second writer exists next to whatever the application writes from its API. JPA saves the
whole row, so the branch committing second puts back what it read at its start and the other
branch's work is gone, without an exception and without a log line.

Pick a strategy and say which one:

- an entity per phase in a 1:1 relation to the aggregate,
- `@DynamicUpdate` while the branches write different attributes,
- `@Version` plus a retry,
- a relation of its own that is only ever appended to.

The blueprint `persistence-parallel-branches` shows the first and explains when the others fit.

## 8. One primary BPMN process per workflow aggregate

VanillaBP builds one `ProcessService` per workflow aggregate class, which is what
`ProcessService<Ride>` injects, so exactly one BPMN process is the one `startWorkflow` starts.
Two classes declaring a different `bpmnProcess` for the same aggregate would make that a coin
flip, so VanillaBP refuses to start and names both classes and both processes.

When a second process works on the same aggregate, declare it in `secondaryBpmnProcesses` of the
class carrying the primary one. A process called by a call activity is the typical case.

Several classes annotated with `@WorkflowService` for one aggregate are allowed as long as each
of them declares the same `bpmnProcess`. That is what makes the split below possible.

This rule is easy to break when two blueprints are composed, because each brings a handler class
declaring a `bpmnProcess` of its own.

### Splitting a large workflow

A large process makes both `Service` and `WorkflowTaskHandler` large, and a single pair of
classes for thirty tasks helps nobody. Large processes have sections, and the model usually says
where they are: an embedded subprocess, a group, or the status transitions the process moves
through.

Give each section a Java package of its own with its own `Service` and its own
`WorkflowTaskHandler`. Every one of those handler classes declares the same `bpmnProcess`, so
they stay one workflow with one `ProcessService` while the code follows the structure of the
model.

```
<base-package>.<usecase>
├── model/                       the aggregate, shared by all sections
├── application/                 section: the application is filled in
│   ├── Service.java
│   └── WorkflowTaskHandler.java
└── assessment/                  section: the application is assessed
    ├── Service.java
    └── WorkflowTaskHandler.java
```

### User tasks get a package of their own

A user task is more than a `@WorkflowTask` method. It has an API of its own, because a user has
to be shown the task and has to complete it, and many user tasks carry state of their own that
lives until the task is completed and belongs to nobody else.

So give each user task its own package, holding the classes serving only that task, its
`ApiController` among them.

The blueprint showing this comes with the Business Cockpit and does not exist yet, so treat this
section as guidance and not as something you can copy from a repository.

## 9. Do not copy reference documentation into the generated project

Link it. See [documentation-map.md](documentation-map.md).

## 10. Do not invent BPMS-specific configuration

If something appears to need it, it belongs into the adapter's wiki, not into the application.
Say that you cannot solve the task without BPMS specifics instead of guessing keys.

## And one about transactions

Do not put `@Transactional` on a workflow service. VanillaBP 2 manages the transaction around a
`@WorkflowTask` method itself: it loads the aggregate, invokes the method and saves the
aggregate again. A `TaskException` commits the aggregate changes while completing the task with a
BPMN error. Your own transaction boundary changes those rollback semantics, which is why the
startup check rejects an annotation that joins VanillaBP's transaction without excluding
`TaskException`.

Code calling `ProcessService` still needs a transaction: `startWorkflow`, `completeTask`,
`cancelTask`, `completeUserTask`, `cancelUserTask` and `correlateMessage` require an active one.
The read-only viewer methods do not.
