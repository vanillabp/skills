# Behaviors that differ

Nothing to change in your code, but worth knowing before you go live. Walk through this list
with the user after the upgrade builds.

## New abilities the application may now use

An application can learn that a workflow ended. Version 1 offered nothing for it, so
applications modelled a service task in front of every end event. `@WorkflowEnded` replaces
that, and the old service task keeps working, so this is something to simplify at leisure rather
than an upgrade step.

BPMN signals can be sent. Version 1 had no API for it, so a signal event was only usable between
two elements of the same model. `ProcessService.sendSignal(name)` broadcasts one now, and
because a signal is a broadcast it takes no workflow aggregate.

An application can tell the BPMS that the aggregate changed. Version 1 pushed the aggregate only
at its own sync points, so a conditional event had nothing to react to and a gateway evaluated
whatever the last task completion had written. `ProcessService.aggregateChanged(aggregate)`
pushes on demand and `aggregateChanged(aggregate, taskId)` into the scope of one task instance.
On Camunda 7 this is what makes conditional events usable, and on Camunda 8 it needs a cluster
with secondary storage.

Workflows the BPMS starts on its own now get a workflow aggregate. A timer, signal or conditional
start event could not be used in version 1, because the workflow had no aggregate and nothing
could be routed to it. VanillaBP 2 builds the aggregate itself and an optional
`@WorkflowStartedByBpms` method lets your code have a say. Nothing changes for the workflows you
start yourself.

## Runtime behavior that changed underneath you

Remote BPMS start workflows in two phases. `startWorkflow` no longer talks to the BPMS inside
your transaction: an outbox entry rides your transaction and the instance is created after the
commit. A rolled-back transaction can therefore no longer leave a workflow behind. In exchange
the start is visible in the BPMS a moment later, and it is at-least-once: a duplicate start is
unlikely, since VanillaBP probes before re-dispatching, yet not impossible.

Camunda 7 does it that way as well now. It delivers tasks inside the engine's transaction, but
every operation progressing a workflow runs after your commit, so it can be repeated when it
loses a concurrency conflict. An application on Camunda 7 therefore needs a transaction outbox,
which version 1 did not ask for.

A repeated delivery of a task no longer runs your handler again. Version 1 passed the
at-least-once delivery of remote BPMS straight through, so a redelivery invoked the
`@WorkflowTask` method a second time and every handler had to guard itself. VanillaBP 2 records
what it processed and answers a repeated delivery from that record. The guards in your handlers
stay correct and still cover what a record cannot: two deliveries running at the same time, and
everything a handler does outside its transaction.

There is a uniform error contract for `@WorkflowTask` methods across all BPMS. A normal return
completes the task, a `TaskException` becomes a BPMN error with the aggregate changes committed,
and any other exception rolls the transaction back and leaves retrying to the BPMS.

Operations on existing workflows find their BPMS by asking. `completeTask`, `cancelTask`, the
user task operations and `correlateMessage` probe the prioritized adapters and remember the
answer. A task or workflow no BPMS knows raises a guiding `TaskNotFoundException` or
`WorkflowNotFoundException`, and one already completed makes the operation a warned no-op. While
a BPMS is unreachable the operation fails instead of falling back to another BPMS, because
silently starting to use the wrong BPMS would be worse.

Configuration defects fail at startup, not at runtime, and the messages name the property
keys to add. An application whose adapter section is missing still boots and tells you what to
write, while a genuinely inconsistent section fails the boot.

Process definition ids of the viewer API are namespaced as `<adapter id>#<BPMS specific id>` and
are opaque. Pass them back unchanged and never parse or compose them.

## Two that need a decision

`version` does what version 1 only documented. The attribute of `@WorkflowTask`, and now also of
`@WorkflowStartedByBpms` and `@WorkflowEnded`, exists since version 1 and was never evaluated, so
every method served every version. VanillaBP 2 matches it against the version of the deployed
process definition the BPMS reports, and a boundary may also name a version tag of the model.
For an application which already carries the attribute that has three consequences: methods whose
ranges were meant to be disjoint really are disjoint now, so a version served by none of them
fails the delivery instead of running the first method; a BPMS which reports no version reaches
methods without the attribute only, so a task whose every method names versions fails the
delivery there; and two methods wired to one task with overlapping ranges fail the boot where
version 1 accepted them. Applications without the attribute see no change.

`@SyncWithBPMS` and `@NoSyncWithBPMS` are real now, where version 1 documented but did not ship
them. Without any annotation every adapter shares everything, Camunda 7 included, so your models
keep reading what they read in version 1, now through process variables instead of a live read
into the aggregate. As soon as you annotate one attribute you are in charge, so read
[the sharing rules](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-aggregates#fine-grained-control-over-attributes-synchronized-to-the-bpms)
before annotating, because annotating a single attribute also decides what happens to all the
others.

## Camunda 7 specifically

The live read of the aggregate is a fallback now, and version 2.1 removes it. Version 1 answered
every BPMN expression by reading the aggregate. Version 2 writes the shared values as process
variables and lets the engine resolve them, so workflows which were already running when you
upgraded carry no such variables and the adapter still reads the aggregate where the engine has
none, logging once what it did.

Two cases the sharing rules do not cover at all were legal in version 1: an attribute readable
only as a field, and an `isX()` method returning something other than `boolean`. Give those a
getter, either `getX()` or an `isX()` returning `boolean`, and share it. While your application
starts, the Camunda 7 adapter lists every expression which needs attention, and that list is your
migration backlog.

## Camunda 8 specifically

On 8.8 Camunda 8 cannot cancel a Camunda-managed user task by BPMN error, which is what
`cancelUserTask` does, because the engine has no such command. A guiding error explains it.
Model the error path explicitly until Camunda's listener support arrives.
