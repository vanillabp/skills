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

## 8. One `@WorkflowService` class per workflow aggregate class

When a second process works on the same aggregate, name it in `secondaryBpmnProcesses` of the
existing class and put its `@WorkflowTask` methods there. Do not annotate a second class with
`@WorkflowService` for that aggregate: VanillaBP builds one `ProcessService` per aggregate class
and starts the process of whichever class the classpath scan found first, so `startWorkflow` may
start the wrong process. Nothing says so, the workflow runs and the aggregate ends up half
filled.

This is easy to walk into when two blueprints are composed, because each brings a handler class
of its own.

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
