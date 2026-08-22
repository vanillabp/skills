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

### Check first whether the class is also the business service

Small projects often put everything in one class: the `@WorkflowTask` methods and the business
methods the API calls sit side by side. Deleting the annotation there takes the transaction away
from the business methods as well, and they still need one. VanillaBP wraps `@WorkflowTask`
methods only, and every call into `ProcessService` requires an active transaction, so a
`startWorkflow` from an endpoint fails once the annotation is gone.

Nothing reports this. The application starts, the workflows the BPMS drives keep working, and
only the paths entered from the API break.

So decide per class:

- Split it along the reference structure, which is the way the blueprints do it: the
  `@WorkflowTask` methods into `WorkflowTaskHandler`, the business code into `Service`, the calls
  into `ProcessService` into `Workflow`. Only the first of those loses its annotation.
- Or keep `@Transactional(noRollbackFor = TaskException.class)` on the combined class. The
  startup check accepts it, the business methods keep their transaction and the `@WorkflowTask`
  contract stays intact.

Deleting the annotation while leaving the class combined is the one option that is wrong.

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

## Hand-written `ProcessService` implementations

`ProcessService` gained methods and made them `default`, so this does not break again.
`getProcessDefinitions`, `getBpmnXml`, `getWorkflowHistory` and `startWorkflowByMessage(A, String)`
are implemented by a VanillaBP adapter, so test doubles keep compiling and only
`getWorkflowModuleId()` stays abstract.

Custom aggregate persistence is not part of the upgrade. Version 1 offered no SPI for it, so a
version 1 project persists through Spring Data with JPA or MongoDB and version 2 does the same
out of the box. See [dependencies.md](dependencies.md).
