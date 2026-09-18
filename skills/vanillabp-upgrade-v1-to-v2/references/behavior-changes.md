# Behaviors that differ

Nothing to change in your code, but worth knowing before you go live. Walk through this list
with the user after the upgrade builds.

## New abilities the application may now use

An application can learn that a workflow ended. Version 1 offered nothing for it, so
applications modelled a service task in front of every end event. `@WorkflowEnded` replaces
that, and the old service task keeps working, so this is something to simplify at leisure rather
than an upgrade step. On Camunda 8, tell the user to wait until the workflows they upgraded have
ended before deleting that service task. The notification hangs off an execution listener the
adapter writes into the model it deploys, and a workflow which was already running stays on the
definition version it was started on, so it never triggers one. Camunda 7 has no such wait,
because it attaches the same listener while it parses a process definition, which reaches every
version its engine holds.

BPMN signals can be sent. Version 1 had no API for it, so a signal event was only usable between
two elements of the same model. `ProcessService.sendSignal(name)` broadcasts one now, and
because a signal is a broadcast it takes no workflow aggregate.

An application can tell the BPMS that the aggregate changed. Version 1 pushed the aggregate only
at its own sync points, so a conditional event had nothing to react to and a gateway evaluated
whatever the last task completion had written. `ProcessService.aggregateChanged(aggregate)`
pushes on demand and `aggregateChanged(aggregate, taskId)` into the scope of one task instance.
On Camunda 7 this is what makes conditional events usable, and on Camunda 8 it needs a cluster
with secondary storage.

Decision tables travel with the workflow module. A `.dmn` file at the same location as the BPMN
files is deployed with the module, to the same BPMS, in the same deployment and into the same
tenant, so a business rule task finds its decision without a second mechanism. Version 1 had no
such thing, so an application which uses decision tables deploys them by hand or by a mechanism of
its own, and after the upgrade two things write the same decision. Where both write the same bytes
nothing happens at all, because both Camunda engines answer a redeployment of an unchanged file by
keeping what they have. Where they write different versions, one of the two has to go. An
application with no `.dmn` file at the module's resources location notices nothing.

Under `name-clash-avoidance: use-prefix` there is more to it. The Camunda adapters rewrite the
decision ids of the files the module deploys the way they rewrite its process ids, and they rewrite
the reference of the business rule task with them. A business rule task pointing at a decision this
module does not deploy then breaks, because the reference is rewritten and the decision in the BPMS
is not. Deploy that decision with the module, or keep the module apart by a tenant, which is the
default. See
[decision tables](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-modules#decision-tables-travel-with-the-processes-calling-them).

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
which version 1 did not ask for. Forgetting it is not a production surprise: where the first
adapter in the priority list needs a two-phase commit and no store resolves, the application
does not boot and the message names what to add.

A repeated delivery of a task no longer runs your handler again. Version 1 passed the
at-least-once delivery of remote BPMS straight through, so a redelivery invoked the
`@WorkflowTask` method a second time and every handler had to guard itself. VanillaBP 2 records
what it processed and answers a repeated delivery from that record. The guards in your handlers
stay correct and still cover what a record cannot: two deliveries running at the same time, and
everything a handler does outside its transaction.

There is a window right after the upgrade where it does not hold, and the guards are what carries
it. A task which was open when the application was stopped has no record, so the first redelivery
after the upgrade runs the handler once more, exactly as version 1 always did. Nothing can be done
about it: an activated Camunda 8 job which is still there may be a handler waiting for its
`completeTask` or a handler which crashed halfway, the cluster cannot tell the two apart, and a
record written for the second case would skip business code which never ran. Tell the user to keep
the version 1 guards until the tasks which were open at the upgrade have all been delivered once,
and point them at the startup: where the BPMS holds tasks open which VanillaBP has no record of, it
says so once per BPMN process with the number, and that number falls to zero as each of them is
delivered once. Camunda 7 is not affected at all, because it delivers inside the engine's
transaction and reports no delivery identity, so it has no such record to be missing.

There is a uniform error contract for `@WorkflowTask` methods across all BPMS. A normal return
completes the task, a `TaskException` becomes a BPMN error with the aggregate changes committed,
and any other exception rolls the transaction back and leaves retrying to the BPMS.

A `@TaskParam` gets the number the BPMS reported or no number at all. Version 1 bound the
parameter by raw reflection, so a value reflection could not pass threw `IllegalArgumentException:
argument type mismatch`, with nothing in the message to work from. VanillaBP 2 converts the value
into the declared type, and since 2026-09-16 it converts a number only where the conversion keeps
it: the value travels through its decimal form and is delivered only where it reads back as the
same number. So a `BigDecimal` of `120.50` still reaches a `Double` parameter as `120.5`, because
the two are the same number, while a `Long` of `3000000000` bound to an `int` ends the task instead
of arriving as `-1294967296`. The message names the value, the declared type and the number which
would have arrived. The same holds for an attribute of an aggregate a BPMS-initiated start writes,
which goes through the same conversion.

For an application coming from version 1 this is a return rather than a break: reflection refused
every one of these pairs too. What it did accept, a `Long` and an `Integer` into a `long`, is
accepted now as well. In between the two versions there were VanillaBP 2 snapshots which converted
such a pair silently, so an application which tested against one of those may have a handler which
has been working on a number nobody wrote. That is what the survey of step 0 looks for. There is
no property which switches the refusal off.

A `@TaskParam` may also be declared as the type the aggregate holds. The platform shares an enum as
the name of its constant and a value type as the text it travels as, and since 2026-09-16 the way
back reads exactly those texts: a `UUID`, an enum, the `java.time` values `Instant`, `LocalDate`,
`LocalTime`, `LocalDateTime`, `OffsetDateTime`, `OffsetTime`, `ZonedDateTime`, `Year`, `YearMonth`,
`MonthDay`, `Duration`, `Period`, `ZoneId` and `ZoneOffset`, and since 2026-09-17 a
`java.util.Date` and a `java.util.TimeZone` as well. Version 1 threw the same
`argument type mismatch` for every one of them, so a handler which declares a `String` and parses it
itself is the workaround an application is likely to carry, and it keeps working. The list is a
selection rather than a promise about the JDK, so a type somebody misses is worth an issue.

A `java.util.Calendar` and a `java.util.Locale` are refused, and the message names the type to
declare instead. A text a type does not travel as is refused too, with an example of one it does,
and a constant name the model invented is refused naming the constants the enum has.

The text of a `java.util.Date` in the BPMS changed with that, and it is the one item here which is
visible in the model. Version 1 and the VanillaBP 2 snapshots before 2026-09-17 shared such an
attribute as `Wed Sep 16 21:55:30 CEST 2026`, and it is shared as `2026-09-16T19:55:30.123Z` now.
The milliseconds survive and the text no longer depends on the server which wrote it, while an
operator reads UTC rather than local time. Ask the user whether a BPMN expression of theirs compares
that text, because such an expression reads something else from now on. A `java.util.TimeZone`
attribute is a fix rather than a change: it ended every sync point with an
`InaccessibleObjectException` before, and it travels as the id of its zone now.

An attribute whose text nothing reads back is named while the application boots, once per attribute,
with the type to share instead. That one is a warning and the application boots.

A `java.util.Calendar` attribute is the exception, since 2026-09-17. An aggregate which shares one
does not start at all, because the text such an attribute wrote is the debug form of the
implementation, 769 characters naming every field of it, and no model reads a point in time out of
it. The message names the attribute, the class it belongs to and the type it is declared as, and it
says to share an `Instant` instead, with a `TimeZone` or a `ZoneId` next to it where the zone
matters too. Where no model needs the attribute, `@NoSyncWithBPMS` says so and the application
starts. A `Calendar` in a nested object or in a collection is refused the same way. Version 1 wrote
the dump into the BPMS, so nothing of value is lost by the change.

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

## What needs a decision from you

An aggregate which shares everything does not start, and this is the one item here which stops your
first boot. Version 1 shipped no sync model, so an application which never annotated anything now
hands every attribute it reaches to the BPMS, at every sync point, and on a remote BPMS those
values leave the application. Nobody chose that, which is why it is said out loud once instead of
being written into a log. The message names the workflow, the aggregate and the attributes, and it
offers two ways on. Say what the models really need, with `@NoSyncWithBPMS` on the class and
`@SyncWithBPMS` on the attributes an expression reads. Or allow the full sync for that one
workflow:

```yaml
vanillabp:
  workflow-modules:
    <workflow-module>:
      workflows:
        <bpmn-process-id>:
          allow-full-sync-with-bpms: true
```

The permission belongs to the single workflow and is not inherited, so the same line one level up
does nothing. It is meant that way: the next workflow somebody adds would be allowed as well, and
nobody looked at that one. Read
[the sharing rules](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-aggregates#fine-grained-control-over-attributes-synchronized-to-the-bpms)
with the user before deciding, because both ways are a decision about data leaving the application.

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

`@BpmnProcess` carries the same attribute, and it is the fallback of the three method annotations
now. A method naming no range of its own serves the range its `@BpmnProcess` declares, on the
primary process and on a secondary one alike. This attribute has existed since version 1 too and
nothing ever read it, so an application which wrote it believing it worked had every method serving
every version, and now has methods which stop being called, without an error. The default is `*`
and leaves everything as it was, so an application which never wrote it is unaffected. A range a
method inherits restricts it the same way a range of its own would: where the BPMS cannot report a
version, only a method whose class names none either is still reached. Every message about a range
says which declaration it came from.

`@SyncWithBPMS` and `@NoSyncWithBPMS` are real now, where version 1 documented but did not ship
them. Without any annotation every adapter shares everything, Camunda 7 included, so your models
keep reading what they read in version 1, now through process variables instead of a live read
into the aggregate. That is also the state the first item of this section refuses, so a version 1
application meets both halves of this rule at once. As soon as you annotate one attribute you are
in charge, so read
[the sharing rules](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-aggregates#fine-grained-control-over-attributes-synchronized-to-the-bpms)
before annotating, because annotating a single attribute also decides what happens to all the
others.

## Camunda 7 specifically

The live read of the aggregate is a fallback now. Version 1 answered every BPMN expression by
reading the aggregate. Version 2 writes the shared values as process variables and lets the engine
resolve them, so workflows which were already running when you upgraded carry no such variables
and the adapter still reads the aggregate where the engine has none. It reports every name it
served that way, once per name, and it counts the workflows which still depend on one. The count
falls on its own as those workflows reach their next sync point, and the fallback is meant to go
away once it carries nobody. It also answers a top-level name and nothing else, so an expression
reading past the first dot is on the process variables from the first day.

Two cases the sharing rules do not cover at all were legal in version 1: an attribute readable
only as a field, and an `isX()` method returning something other than `boolean`. Give those a
getter, either `getX()` or an `isX()` returning `boolean`, and share it. While your application
starts, the Camunda 7 adapter names the expressions whose value the engine does not hold, with the
element, the segment the path stops at and what this engine does with the resulting null.

That report is the backlog, and it is not the whole of it. The check reads declared types, so it
says nothing where a path runs through a `Map`, an interface or a collection without an element
type, and it judges no method call, which leaves `${order.getTotal()}` and `${order.status.name()}`
out although both stop working. Those two raise an incident, so the engine tells you about them.
The silent half is what the check exists for, a conditional event above all, which answers a
condition it cannot evaluate with false and then waits for good.

An expression reading a nested value needs one setting. Camunda 7 stores a string, a number it has
a type for, a boolean, a date and bytes as themselves, and everything else as an object variable,
in whatever serialization format the engine was told to use. A nested value is such a variable, and
so is a `BigDecimal`, a `BigInteger` or a `Float`. Without
`vanillabp.adapters.<id>.serialization-format` and a dataformat plugin on the classpath the engine
falls back to Java serialization: Cockpit shows a blob instead of the data, the engine's database
holds serialized instances of your classes, and `${order.customer.name}` works only while the class
in the database and the class on the classpath still match. The adapter warns once when it writes
such a value without a format. Version 1 never had this question, because it handed the expression
the live Java object.
[What becomes of a shared value](https://github.com/camunda-community-hub/vanillabp-camunda7-adapter/wiki/What-becomes-of-a-shared-value)
has the formats, the plugin and what each format cannot carry.

## Camunda 8 specifically

A failed job waits before the cluster hands it out again. Version 1 sent a backoff only where the
application configured one or the model carried a `retryBackoff` task header, so a handler failing on
something which needs a moment burned its three retries as fast as the cluster could redeliver.
Version 2 defaults `vanillabp.adapters.<id>.retry-backoff` to `PT10S`, still resolvable per workflow
module, workflow and task.

The `retryBackoff` task header is read as before, so a model which carries one needs no change. Two
details differ. Version 1 looked at the header only where the element's `zeebe:taskDefinition` also
carried a `retries` attribute, and that condition is gone. And version 2 has the property down to the
single task, so the two can meet: where a task is configured AND modelled, the configured value of that
one task wins, one line per element says so, and every level above the task loses to the model. A header
which is no ISO-8601 duration warns once per element and leaves the configured value in force, where
version 1 fell back to a backoff of zero.

A start waits for its cluster. The adapter asks the cluster for its topology while the application
boots and gives it ten minutes to answer, `vanillabp.adapters.<id>.startup-wait`. A deployment
pipeline which expected a start against an absent cluster to fail within seconds now waits those
ten minutes out, so a pipeline with a timing assumption is the case to look at. `PT0S` is the old
behaviour. Nothing waits silently: a line before the first attempt names the address and how long
the adapter will wait, and a line every few seconds says how much of it is gone. An answer the
cluster will repeat, a refused request for instance, ends the start at once instead of waiting.
`401` is deliberately not such an answer, because the client refreshes an expired token. The key
belongs to the adapter alone, since what is waited for is a cluster and a cluster belongs to no
workflow module.

Two checks on `request-timeout` come with it. A value which is not positive ends the boot, because
it is the deadline of every request the adapter sends. A value below one second is warned about and
the boot goes on.

On 8.8 Camunda 8 cannot cancel a Camunda-managed user task by BPMN error, which is what
`cancelUserTask` does, because the engine has no such command. A guiding error explains it.
Model the error path explicitly until Camunda's listener support arrives.

The workflows the user upgraded already carry their process variables, unlike on Camunda 7:
version 1 handed the whole workflow aggregate to every command it sent, so an instance started
under version 1 has a variable for everything its JSON serialization could read, the ID
attribute included, which is what version 2's probes look for. Two cases break that and both are
in the user's code:

- the aggregate's ID attribute renamed for JSON (`@JsonProperty` on the ID). Version 1 wrote the
  variable under the JSON name, version 2 looks for the name the persistence layer reports, and
  every probe for a pre-upgrade instance then finds nothing;
- an attribute version 1 serialized as a FIELD. Version 2 reads getters only, so the variable
  keeps the last value version 1 wrote for good. That is a stale value rather than a missing one
  and nothing reports it. Give the attribute a getter and share it.

Check both while surveying the project, because neither shows up as an error.
