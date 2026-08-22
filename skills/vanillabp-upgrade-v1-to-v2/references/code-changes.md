# Code changes

Four of them, and the compiler finds the first three once the dependencies are swapped.

## Removed `ProcessService` overloads

The overloads taking a message object, whose class simple name became the message name, are
gone. Message content never travels to the BPMS, so an object serving only as a name carrier
earned no API.

```java
// before
processService.correlateMessage(ride, rideConfirmation);
processService.correlateMessage(ride, rideConfirmation, correlationId);
processService.startWorkflowByMessage(ride, rideRequested);

// after: name the message explicitly, formerly message.getClass().getSimpleName()
processService.correlateMessage(ride, "RideConfirmation");
processService.correlateMessage(ride, "RideConfirmation", correlationId);
processService.startWorkflowByMessage(ride, "RideRequested");
```

Incorporate the message's data into the aggregate before correlating. As before, the aggregate
is the single source of truth.

## `@BpmnProcess.primary()` is gone

Whether a process is the primary one follows from where it is declared. `bpmnProcess` is
primary, `secondaryBpmnProcesses` are not.

```java
// before
@WorkflowService(workflowAggregateClass = Ride.class,
    bpmnProcess = @BpmnProcess(bpmnProcessId = "RideProcess", primary = true),
    secondaryBpmnProcesses = @BpmnProcess(bpmnProcessId = "RideCancellation", primary = false))
// after
@WorkflowService(workflowAggregateClass = Ride.class,
    bpmnProcess = @BpmnProcess(bpmnProcessId = "RideProcess"),
    secondaryBpmnProcesses = @BpmnProcess(bpmnProcessId = "RideCancellation"))
```

`secondaryBpmnProcesses` is honored now, where the version 1 platform integration ignored it, so
every declared BPMN process is wired for task processing. Secondary entries must name an
explicit `bpmnProcessId`.

A version 1 application which spread the processes of one workflow aggregate over several
classes, each with its own `bpmnProcess`, has to move them together: one class declares the
process that `startWorkflow` starts, the others belong into its `secondaryBpmnProcesses`.
Handlers may still live in separate classes as long as each of them declares that same
`bpmnProcess`. Version 2 says so while starting, on Quarkus while building, instead of picking
one of the processes by the order the classes were found in.

## Remove `@Transactional` from workflow services

Version 1 asked you to write `@Transactional(noRollbackFor = TaskException.class)` on workflow
services. VanillaBP 2 manages the transaction around a `@WorkflowTask` method itself: it loads
the aggregate, invokes the method and saves the aggregate again. That includes the same
contract: a `TaskException` commits the aggregate changes and completes the task with a BPMN
error.

```java
// before
@Service
@WorkflowService(workflowAggregateClass = Ride.class, ...)
@Transactional(noRollbackFor = TaskException.class)
public class RideService { ... }

// after: no transaction annotation of your own
@Service
@WorkflowService(workflowAggregateClass = Ride.class, ...)
public class RideService { ... }
```

Keeping the version 1 annotation is valid too. The startup check accepts a rollback rule that
excludes `TaskException`, so the version 1 line boots and behaves as before. What fails the boot
is an annotation joining VanillaBP's transaction without such a rule, because a `TaskException`
would then discard the aggregate changes instead of committing them.

The check also reports an annotation inherited from a superclass or an interface and one hidden
inside a custom annotation of yours. If the annotation sits on a bean the handler calls, no
startup check can see it, and the task fails at runtime with a message naming the workflow, the
task and the same two remedies.

A `javax.transaction.Transactional` carried over from Spring Boot 2 is honored by neither Spring
Framework 7 nor Quarkus 3. It does not break the contract, it simply does nothing, and VanillaBP
warns about it at startup so the transaction boundary you believe in does not stay imaginary.

Code calling `ProcessService` still needs a transaction: `startWorkflow`, `completeTask`,
`cancelTask`, `completeUserTask`, `cancelUserTask` and `correlateMessage` require an active one.
The read-only viewer methods do not.

## Hand-written `ProcessService` and `AggregatePersistenceAware` implementations

Both interfaces gained methods and both made them `default`, so this does not break again.

`ProcessService`: `getProcessDefinitions`, `getBpmnXml`, `getWorkflowHistory` and
`startWorkflowByMessage(A, String)` are `default`, implemented by a VanillaBP adapter. Test
doubles keep compiling, and only `getWorkflowModuleId()` stays abstract.

`AggregatePersistenceAware`: all methods are `default` with guiding messages. Implement
`loadById(Object)`, because VanillaBP loads aggregates itself now for task processing and
aggregate sync. If your aggregate's ID does not convert losslessly from and to `String`, either
return the ID type from `getAggregateIdType()` or return `null` there to own the serialized form
yourself. Remote BPMS also need `getAggregateIdName()`.

The platform-provided implementations cover all of this, so this concerns your own ones only. On
Quarkus that circle is wider now than in version 1: an aggregate managed by a Panache repository,
written as a Panache active record for JPA or MongoDB, or managed by a Spring Data repository is
persisted by VanillaBP itself, so the `AggregatePersistenceAware` a version 1 Quarkus application
had to write for exactly that can be deleted. Keep it where it does more than store and load, and
keep in mind that your own implementation always wins over the built-in one.
