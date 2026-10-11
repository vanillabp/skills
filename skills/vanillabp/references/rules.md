# Rules that hold regardless of blueprint

If you break one of these, a VanillaBP application stops being one, without a word. They repeat
`https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`, which wins
where the two disagree.

## 1. No process variables

All state lives in the workflow aggregate. Sequence flow conditions call methods on the
aggregate instead of reading variables.

## 2. No BPMS API in application code

No `RuntimeService`, no `ZeebeClient`, no `CamundaClient`, no other engine class. The only
compile-time dependency on the BPMS (business process management system) is the VanillaBP SPI
(service provider interface). If you use an engine API, the hexagonal architecture is broken,
and the application can no longer change its BPMS.

## 3. No BPMS dependency in a workflow module

A pure workflow module depends on `io.vanillabp:vanillabp-spring-boot-support` or
`io.vanillabp:vanillabp-quarkus-support`, and never on an adapter. Those modules do not expose
BPMS APIs, on purpose. The adapter dependency belongs in the application module.

## 4. No business state in the BPMN

The model expresses the flow, not the data. Nothing an auditor would want to read belongs in
the diagram.

## 5. One aggregate per workflow

Do not share an aggregate between two processes, unless the blueprint
`bpmn-call-activity-decomposition` shows it explicitly. Do not store two business objects in
one aggregate.

## 6. No direct message exchange between workflow modules

A process must never send a message to a process of another workflow module, because that
couples them. Use the way `module-interaction` shows: an outgoing message becomes an internal
API call, and the receiving module delivers a message through its own API.

## 7. Do not let two branches of one workflow write the aggregate carelessly

A parallel gateway, a non-interrupting boundary event and a multi-instance task each add a
token. A token marks where a process instance currently is. So a second writer exists next to
whatever the application writes from its API. JPA (Jakarta Persistence) saves the whole row. So
the branch which commits second puts back what it read at its start, and the other branch's
work is gone, without an exception and without a log line.

Pick a strategy and say which one:

- an entity per phase in a 1:1 relation to the aggregate,
- `@DynamicUpdate` while the branches write different attributes,
- `@Version` plus a retry,
- a relation of its own which is only ever added to.

The blueprint `persistence-parallel-branches` shows the first, and explains when the others
fit.

## 8. One primary BPMN process per workflow aggregate

VanillaBP builds one `ProcessService` per workflow aggregate class, and that is what
`ProcessService<Ride>` injects. So exactly one BPMN process is the one `startWorkflow` starts.
If two classes declared a different `bpmnProcess` for the same aggregate, that would be a coin
flip. So VanillaBP refuses to start, and names both classes and both processes.

When a second process works on the same aggregate, declare it in `secondaryBpmnProcesses` of
the class which carries the primary one. A process called by a call activity is the typical
case.

Several classes annotated with `@WorkflowService` for one aggregate are allowed, as long as
each of them declares the same `bpmnProcess`. That makes the split below possible.

This rule breaks easily when you combine two blueprints, because each brings a handler class
which declares a `bpmnProcess` of its own.

### Splitting a large workflow

A large process makes both `Service` and `WorkflowTaskHandler` large. A single pair of classes
for thirty tasks helps nobody. Large processes have sections, and the model usually says where
they are: an embedded subprocess, a group, or the status changes the process goes through.

Give each section a Java package of its own, with its own `Service` and its own
`WorkflowTaskHandler`. Every one of those handler classes declares the same `bpmnProcess`. So
they stay one workflow with one `ProcessService`, while the code follows the structure of the
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
to see the task and has to complete it. Many user tasks also carry state of their own, which
lives until the task is completed and belongs to nobody else.

So give each user task its own package. It holds the classes which serve only that task, its
`ApiController` among them.

The blueprint which shows this comes with the Business Cockpit, and it does not exist yet. So
treat this section as guidance, not as something you can copy from a repository.

## 9. Do not copy reference documentation into the generated project

Link it. See [documentation-map.md](documentation-map.md).

## 10. Do not invent BPMS-specific configuration

If something seems to need it, it belongs in the adapter's wiki, not in the application. Say
that you cannot solve the task without details of the BPMS, instead of guessing keys.

## And one about transactions

Do not put `@Transactional` on a workflow service. VanillaBP 2 manages the transaction around a
`@WorkflowTask` method itself. It loads the aggregate, calls the method and saves the aggregate
again. A `TaskException` commits the changes of the aggregate, while it completes the task with
a BPMN error. A transaction border of your own changes what is rolled back. That is why the
check at startup rejects an annotation which joins VanillaBP's transaction without excluding
`TaskException`.

Code which calls `ProcessService` still needs a transaction. `startWorkflow`, `completeTask`,
`cancelTask`, `completeUserTask`, `cancelUserTask` and `correlateMessage` require an active
one. The viewer methods, which only read, do not.
