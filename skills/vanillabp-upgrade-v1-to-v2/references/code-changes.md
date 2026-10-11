# Code changes

There are four of them. Once the dependencies are swapped, the compiler finds the first three.

## Removed `ProcessService` overloads

The overloads which take a message object are gone. There the simple name of the object's class
became the message name. Message content never travels to the BPMS (business process management
system). So an object which only carries a name does not get an API.

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

Put the message's data into the aggregate before you correlate. As before, the aggregate is the
single source of truth.

## `@BpmnProcess.primary()` is gone

Where a process is declared decides whether it is the primary one. `bpmnProcess` is primary,
`secondaryBpmnProcesses` are not.

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

`secondaryBpmnProcesses` is honored now. The platform integration of version 1 ignored it. So
every declared BPMN process is wired for task processing. Secondary entries must name an
explicit `bpmnProcessId`.

A version 1 application may have spread the processes of one workflow aggregate over several
classes, each with its own `bpmnProcess`. It has to move them together. One class declares the
process which `startWorkflow` starts. The others belong in its `secondaryBpmnProcesses`.
Handlers may still live in separate classes, as long as each of them declares that same
`bpmnProcess`. Version 2 says so while starting, on Quarkus while building. It does not pick
one of the processes by the order in which the classes were found.

## Remove `@Transactional` from workflow services

Version 1 asked you to write `@Transactional(noRollbackFor = TaskException.class)` on workflow
services. VanillaBP 2 manages the transaction around a `@WorkflowTask` method itself. It loads
the aggregate, calls the method and saves the aggregate again. The contract stays the same: a
`TaskException` commits the changes of the aggregate and completes the task with a BPMN error.

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

The `@Service` in both examples now does work. Version 2 no longer scans the classpath for
`@WorkflowService`. On Spring Boot it goes through the bean definitions and keeps the classes
which carry the annotation. Quarkus wants the same at build time. So a workflow service which
is no bean loses its `ProcessService`. A `@Bean` method has to declare the workflow service
class as its return type, because VanillaBP reads the type of the definition, never the
instance. Spring Boot does not fail over it. The application starts, and the deployment warns
about the BPMN processes nothing claims. It names the class and the ways to make it a bean.
Quarkus fails the build instead. While you are in these files anyway, check every class the
survey found.

Keeping the version 1 annotation is valid too. The check at startup accepts a rollback rule
which excludes `TaskException`. So the version 1 line boots and behaves as before. An
annotation which joins VanillaBP's transaction without such a rule fails the boot. With it, a
`TaskException` would throw away the changes of the aggregate instead of committing them.

### Check first whether the class is also the business service

Small projects often put everything in one class. The `@WorkflowTask` methods and the business
methods the API calls sit side by side. Deleting the annotation there takes the transaction
away from the business methods as well, and they still need one. VanillaBP wraps
`@WorkflowTask` methods only, and every call into `ProcessService` requires an active
transaction. So a `startWorkflow` from an endpoint fails once the annotation is gone.

Nothing reports this. The application starts, and the workflows the BPMS drives keep working.
Only the paths which start at the API break.

So decide per class:

- Split it along the reference structure, the way the blueprints do it: the `@WorkflowTask`
  methods into `WorkflowTaskHandler`, the business code into `Service`, the calls into
  `ProcessService` into `Workflow`. Only the first of those loses its annotation.
- Or keep `@Transactional(noRollbackFor = TaskException.class)` on the combined class. The
  check at startup accepts it, the business methods keep their transaction, and the
  `@WorkflowTask` contract stays intact.

The one wrong option is to delete the annotation and leave the class combined.

The check also reports an annotation inherited from a superclass or an interface, and one
hidden inside a custom annotation of yours. If the annotation sits on a bean the handler calls,
no check at startup can see it. Then the task fails at runtime, with a message which names the
workflow, the task and the same two remedies.

Neither Spring Framework 7 nor Quarkus 3 honors a `javax.transaction.Transactional` carried
over from Spring Boot 2. It does not break the contract. It simply does nothing. VanillaBP
warns about it at startup, so that you do not rely on a transaction border which does not
exist.

Code which calls `ProcessService` still needs a transaction. `startWorkflow`, `completeTask`,
`cancelTask`, `completeUserTask`, `cancelUserTask` and `correlateMessage` require an active
one. The viewer methods, which only read, do not.

## Hand-written `ProcessService` implementations

`ProcessService` gained methods and made them `default`, so this does not break again. A
VanillaBP adapter implements `getProcessDefinitions`, `getBpmnXml`, `getWorkflowHistory` and
`startWorkflowByMessage(A, String)`. So test doubles keep compiling, and only
`getWorkflowModuleId()` stays abstract.

A persistence of your own for the aggregate is not part of the upgrade. Version 1 offered no
SPI for it. So a version 1 project stores through Spring Data, with JPA or MongoDB, and version
2 does the same out of the box. See [dependencies.md](dependencies.md).
