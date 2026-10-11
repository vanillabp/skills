# Behaviors that differ

Nothing to change in your code, but worth knowing before you go live. Go through this list with
the user after the upgrade builds.

## New abilities the application may now use

An application can learn that a workflow ended. Version 1 offered nothing for it, so
applications modelled a service task in front of every end event. `@WorkflowEnded` replaces
that, and the old service task keeps working. So this is something to simplify when there is
time, not an upgrade step. On Camunda 8, tell the user to wait until the workflows they
upgraded have ended, before they delete that service task. The notification comes from an
execution listener which the adapter writes into the model it deploys. A workflow which was
already running stays on the definition version it was started on, so it never triggers one.
Camunda 7 has no such wait. It attaches the same listener while it parses a process definition,
and that reaches every version its engine holds.

BPMN signals can be sent. Version 1 had no API for it, so a signal event only worked between
two elements of the same model. `ProcessService.sendSignal(name)` now broadcasts one. A signal
is a broadcast, so it takes no workflow aggregate.

An application can tell the BPMS (business process management system) that the aggregate
changed. Version 1 pushed the aggregate only at its own sync points. So a conditional event had
nothing to react to, and a gateway evaluated whatever the last task completion had written.
`ProcessService.aggregateChanged(aggregate)` pushes on demand, and
`aggregateChanged(aggregate, taskId)` pushes into the scope of one task instance. On Camunda 7
this makes conditional events usable. On Camunda 8 it needs a cluster with secondary storage.

Decision tables travel with the workflow module. A `.dmn` file at the same location as the BPMN
files is deployed with the module: to the same BPMS, in the same deployment, and into the same
tenant. So a business rule task finds its decision without a second mechanism. Version 1 had no
such thing. So an application which uses decision tables deploys them by hand or by a mechanism
of its own. After the upgrade, two things write the same decision. Where both write the same
bytes, nothing happens at all. Both Camunda engines keep what they have when an unchanged file
is deployed again. Where they write different versions, one of the two has to go. An
application with no `.dmn` file at the module's resources location notices nothing.

Under `name-clash-avoidance: use-prefix` there is more to it. The Camunda adapters rewrite the
decision ids of the files the module deploys, the same way they rewrite its process ids. They
rewrite the reference of the business rule task with them. So a business rule task which points
at a decision this module does not deploy breaks. The reference is rewritten, and the decision
in the BPMS is not. Deploy that decision with the module, or keep the module apart by a tenant,
which is the default. See
[decision tables](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-modules#decision-tables-travel-with-the-processes-calling-them).

Workflows which the BPMS starts on its own now get a workflow aggregate. In version 1 you could
not use a timer, signal or conditional start event, because the workflow had no aggregate, and
nothing could be routed to it. VanillaBP 2 builds the aggregate itself. An optional
`@WorkflowStartedByBpms` method lets your code have a say. Nothing changes for the workflows
you start yourself.

## Runtime behavior that changed underneath you

Remote BPMS start workflows in two phases. `startWorkflow` no longer talks to the BPMS inside
your transaction. An outbox entry travels in your transaction, and the instance is created
after the commit. So a rolled-back transaction can no longer leave a workflow behind. In
exchange, the start shows up in the BPMS a moment later, and it happens at least once. A
duplicate start is unlikely, because VanillaBP checks before it dispatches again. But it is not
impossible.

Camunda 7 now does it that way as well. It delivers tasks inside the engine's transaction. But
every operation which moves a workflow on runs after your commit, so it can be repeated when it
loses a conflict with a parallel change. So an application on Camunda 7 needs a transaction
outbox, which version 1 did not ask for. Forgetting it will not surprise you in production.
Where the first adapter in the priority list needs a two-phase commit and no store resolves,
the application does not boot, and the message names what to add.

A repeated delivery of a task no longer runs your handler again. Remote BPMS deliver at least
once, and version 1 passed that straight through. So a second delivery called the
`@WorkflowTask` method a second time, and every handler had to guard itself. VanillaBP 2
records what it processed and answers a repeated delivery from that record. The guards in your
handlers stay correct. They still cover what a record cannot: two deliveries which run at the
same time, and everything a handler does outside its transaction.

Right after the upgrade there is a window where this does not hold, and the guards cover it. A
task which was open when the application was stopped has no record. So the first repeated
delivery after the upgrade runs the handler once more, exactly as version 1 always did. Nothing
can be done about it. An activated Camunda 8 job which is still there may be a handler which
waits for its `completeTask`, or a handler which crashed halfway. The cluster cannot tell the
two apart. A record written for the second case would skip business code which never ran. Tell
the user to keep the version 1 guards until every task which was open at the upgrade has been
delivered once. Point them at the startup. Where the BPMS holds open tasks which VanillaBP has
no record of, it says so once per BPMN process, with the number. That number falls to zero as
each of them is delivered once. Camunda 7 is not affected at all. It delivers inside the
engine's transaction and reports no identity for a delivery, so there is no record which could
be missing.

`@WorkflowTask` methods have one error contract across all BPMS. A normal return completes the
task. A `TaskException` becomes a BPMN error, and the changes of the aggregate are committed.
Any other exception rolls the transaction back, and the BPMS retries.

A `@TaskParam` gets the number the BPMS reported, or no number at all. Version 1 bound the
parameter by plain reflection. So a value which reflection could not pass threw
`IllegalArgumentException: argument type mismatch`, and nothing in the message helped.
VanillaBP 2 converts the value into the declared type. Since 2026-09-16 it converts a number
only where the conversion keeps the value. The value travels through its decimal form, and it
is delivered only where it reads back as the same number. So a `BigDecimal` of `120.50` still
reaches a `Double` parameter as `120.5`, because the two are the same number. But a `Long` of
`3000000000` bound to an `int` ends the task, instead of arriving as `-1294967296`. The message
names the value, the declared type and the number which would have arrived. The same holds for
an attribute of an aggregate which a start triggered by the BPMS writes. It goes through the
same conversion.

For an application which comes from version 1, this brings back the old behaviour and breaks
nothing. Reflection refused every one of these pairs too. What it did accept, a `Long` and an
`Integer` into a `long`, is accepted now as well. Between the two versions there were VanillaBP
2 snapshots which converted such a pair without a word. So an application which tested against
one of those may have a handler which worked on a number nobody wrote. The survey of step 0
looks for that. No property switches the refusal off.

A `@TaskParam` may also be declared as the type the aggregate holds. The platform shares an
enum as the name of its constant, and a value type as the text it travels as. Since 2026-09-16
the way back reads exactly those texts: a `UUID`, an enum, the `java.time` values `Instant`,
`LocalDate`, `LocalTime`, `LocalDateTime`, `OffsetDateTime`, `OffsetTime`, `ZonedDateTime`,
`Year`, `YearMonth`, `MonthDay`, `Duration`, `Period`, `ZoneId` and `ZoneOffset`. Since
2026-09-17 also a `java.util.Date` and a `java.util.TimeZone`. Version 1 threw the same
`argument type mismatch` for every one of them. So an application probably carries a
workaround: a handler which declares a `String` and parses it itself. That keeps working. The
list is a selection, not a promise about the JDK. If somebody misses a type, it is worth an
issue.

A `java.util.Calendar` and a `java.util.Locale` are refused, and the message names the type to
declare instead. A text which a type does not travel as is refused too, with an example of one
it does. A constant name which the model invented is refused, and the message names the
constants the enum has.

The text of a `java.util.Date` in the BPMS changed with that. It is the one item here which you
can see in the model. Version 1 and the VanillaBP 2 snapshots before 2026-09-17 shared such an
attribute as `Wed Sep 16 21:55:30 CEST 2026`. Now it is shared as `2026-09-16T19:55:30.123Z`.
The milliseconds survive, and the text no longer depends on the server which wrote it. But an
operator reads UTC, not local time. Ask the user whether a BPMN expression of theirs compares
that text, because such an expression reads something else from now on. A `java.util.TimeZone`
attribute is a fix, not a change. Before, it ended every sync point with an
`InaccessibleObjectException`. Now it travels as the id of its zone.

An attribute whose text nothing reads back is named while the application boots, once per
attribute, with the type to share instead. That one is a warning, and the application boots.

A `java.util.Calendar` attribute is the exception, since 2026-09-17. An aggregate which shares
one does not start at all. The text such an attribute wrote is the debug form of the
implementation: 769 characters which name every field of it. No model reads a point in time out
of that. The message names the attribute, the class it belongs to and its declared type. It
says to share an `Instant` instead, with a `TimeZone` or a `ZoneId` next to it where the zone
matters too. Where no model needs the attribute, `@NoSyncWithBPMS` says so, and the application
starts. A `Calendar` in a nested object or in a collection is refused the same way. Version 1
wrote that dump into the BPMS, so nothing of value is lost by the change.

Operations on existing workflows find their BPMS by asking. `completeTask`, `cancelTask`, the
user task operations and `correlateMessage` ask the prioritized adapters and remember the
answer. A task or workflow which no BPMS knows raises a `TaskNotFoundException` or
`WorkflowNotFoundException`, which tells you what to do. If it is already completed, the
operation does nothing and logs a warning. While a BPMS cannot be reached, the operation fails
instead of falling back to another BPMS. Starting to use the wrong BPMS without a word would be
worse.

Configuration defects fail at startup, not at runtime, and the messages name the property keys
to add. An application whose adapter section is missing still boots and tells you what to
write. A section which really contradicts itself fails the boot.

Process definition ids of the viewer API have the form `<adapter id>#<BPMS specific id>`, and
they are opaque. Pass them back unchanged. Never parse or build them.

## What needs a decision from you

An aggregate which shares everything does not start. This is the one item here which stops your
first boot. Version 1 shipped no sync model. So an application which never annotated anything
now hands every attribute it reaches to the BPMS, at every sync point. On a remote BPMS those
values leave the application. Nobody chose that. That is why it is said out loud once, instead
of being written into a log. The message names the workflow, the aggregate and the attributes,
and it offers two ways on. Either say what the models really need, with `@NoSyncWithBPMS` on
the class and `@SyncWithBPMS` on the attributes an expression reads. Or allow the full sync for
that one workflow:

```yaml
vanillabp:
  workflow-modules:
    <workflow-module>:
      workflows:
        <bpmn-process-id>:
          allow-full-sync-with-bpms: true
```

The permission belongs to the single workflow and is not inherited. So the same line one level
up does nothing. That is on purpose. Otherwise the next workflow somebody adds would be allowed
as well, and nobody looked at that one. Read
[the sharing rules](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-aggregates#fine-grained-control-over-attributes-synchronized-to-the-bpms)
with the user before you decide. Both ways are a decision about data which leaves the
application.

`version` does what version 1 only documented. The attribute of `@WorkflowTask`, and now also
of `@WorkflowStartedByBpms` and `@WorkflowEnded`, exists since version 1. It was never
evaluated, so every method served every version. VanillaBP 2 matches it against the version of
the deployed process definition which the BPMS reports. A boundary may also name a version tag
of the model. For an application which already carries the attribute, that has three results.
First, methods whose ranges were meant not to overlap really do not overlap now. So a version
which none of them serves fails the delivery, instead of running the first method. Second, a
BPMS which reports no version reaches only methods without the attribute. So a task whose
methods all name versions fails the delivery there. Third, two methods wired to one task with
overlapping ranges fail the boot, where version 1 accepted them. Applications without the
attribute see no change.

`@BpmnProcess` carries the same attribute, and it is now the fallback of the three method
annotations. A method which names no range of its own serves the range its `@BpmnProcess`
declares, on the primary process and on a secondary one alike. This attribute has existed since
version 1 too, and nothing ever read it. So an application which wrote it, believing it worked,
had every method serving every version. Now it has methods which stop being called, without an
error. The default is `*` and leaves everything as it was. So an application which never wrote
it is not affected. A range which a method inherits restricts it the same way a range of its
own would. Where the BPMS cannot report a version, only a method whose class names no range
either is still reached. Every message about a range says which declaration it came from.

`@SyncWithBPMS` and `@NoSyncWithBPMS` really work now. Version 1 documented them but did not
ship them. Without any annotation every adapter shares everything, Camunda 7 included. So your
models keep reading what they read in version 1, now through process variables instead of
reading the aggregate directly. That is also the state which the first item of this section
refuses. So a version 1 application meets both halves of this rule at once. As soon as you
annotate one attribute, you are in charge. So read
[the sharing rules](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-aggregates#fine-grained-control-over-attributes-synchronized-to-the-bpms)
before you annotate. Annotating a single attribute also decides what happens to all the others.

## Camunda 7 specifically

Reading the aggregate directly is now a fallback. Version 1 answered every BPMN expression by
reading the aggregate. Version 2 writes the shared values as process variables and lets the
engine resolve them. Workflows which were already running when you upgraded carry no such
variables. So where the engine has none, the adapter still reads the aggregate. It reports
every name it served that way, once per name, and counts the workflows which still depend on
one. The count falls on its own as those workflows reach their next sync point. The fallback is
meant to go away once nobody depends on it. It also answers a top-level name and nothing else.
So an expression which reads past the first dot uses the process variables from the first day.

Two cases were legal in version 1, and the sharing rules do not cover them at all: an attribute
which can only be read as a field, and an `isX()` method which returns something other than
`boolean`. Give those a getter, either `getX()` or an `isX()` which returns `boolean`, and
share it. While your application starts, the Camunda 7 adapter names the expressions whose
value the engine does not hold. It names the element, the segment where the path stops, and
what this engine does with the resulting null.

That report is your list of open work, but it is not the whole list. The check reads declared
types. So it says nothing where a path runs through a `Map`, an interface or a collection
without an element type. And it judges no method call. That leaves out `${order.getTotal()}`
and `${order.status.name()}`, although both stop working. Those two raise an incident, so the
engine tells you about them. The check exists for the silent half. Above all that is a
conditional event: it answers a condition it cannot evaluate with false, and then waits
forever.

An expression which reads a nested value needs one setting. Camunda 7 stores a string, a number
it has a type for, a boolean, a date and bytes as themselves. It stores everything else as an
object variable, in whatever serialization format the engine was told to use. A nested value is
such a variable, and so is a `BigDecimal`, a `BigInteger` or a `Float`. Without
`vanillabp.adapters.<id>.serialization-format` and a dataformat plugin on the classpath, the
engine falls back to Java serialization. Then Cockpit shows a blob instead of the data, and the
engine's database holds serialized instances of your classes. `${order.customer.name}` works
only while the class in the database and the class on the classpath still match. The adapter
warns once when it writes such a value without a format. Version 1 never had this question,
because it handed the expression the live Java object.
[What becomes of a shared value](https://github.com/camunda-community-hub/vanillabp-camunda7-adapter/wiki/What-becomes-of-a-shared-value)
has the formats, the plugin and what each format cannot carry.

## Camunda 8 specifically

A failed job waits before the cluster hands it out again. Version 1 sent a backoff only where
the application configured one, or where the model carried a `retryBackoff` task header. So a
handler which failed on something that needs a moment used up its three retries as fast as the
cluster could deliver again. Version 2 sets `vanillabp.adapters.<id>.retry-backoff` to `PT10S`
by default. You can still set it per workflow module, workflow and task.

The `retryBackoff` task header is read as before, so a model which carries one needs no change.
Two details differ. Version 1 looked at the header only where the element's
`zeebe:taskDefinition` also carried a `retries` attribute. That condition is gone. And version
2 has the property down to the single task, so the two can meet. Where a task is configured AND
modelled, the configured value of that one task wins, and one line per element says so. Every
level above the task loses to the model. A header which is no ISO-8601 duration warns once per
element, and the configured value stays in force. Version 1 fell back to a backoff of zero
there.

A start waits for its cluster. While the application boots, the adapter asks the cluster for
its topology, and gives it ten minutes to answer: `vanillabp.adapters.<id>.startup-wait`. A
deployment pipeline may expect a start against a missing cluster to fail within seconds. Now it
waits those ten minutes out. So look at pipelines which make such timing assumptions. `PT0S` is
the old behaviour. Nothing waits without a word. A line before the first attempt names the
address and how long the adapter will wait. A line every few seconds says how much of that time
is gone. An answer which the cluster will repeat, a refused request for example, ends the start
at once instead of waiting. `401` is not such an answer, on purpose, because the client
refreshes an expired token. The key belongs to the adapter alone. What it waits for is a
cluster, and a cluster belongs to no workflow module.

Two checks on `request-timeout` come with it. A value which is not positive ends the boot,
because it is the deadline of every request the adapter sends. A value below one second gets a
warning, and the boot goes on.

On 8.8, Camunda 8 cannot cancel a user task which Camunda manages by a BPMN error, and that is
what `cancelUserTask` does. The engine has no such command. An error explains it and tells you
what to do. Model the error path explicitly until Camunda's support for listeners arrives.

Unlike on Camunda 7, the workflows the user upgraded already carry their process variables.
Version 1 handed the whole workflow aggregate to every command it sent. So an instance started
under version 1 has a variable for everything its JSON serialization could read, the ID
attribute included. Version 2 looks for that one when it asks the cluster. Two cases break
that, and both are in the user's code:

- the aggregate's ID attribute renamed for JSON (`@JsonProperty` on the ID). Version 1 wrote
  the variable under the JSON name. Version 2 looks for the name the persistence layer reports.
  Then every search for an instance from before the upgrade finds nothing;
- an attribute which version 1 serialized as a FIELD. Version 2 reads getters only. So the
  variable keeps forever the last value version 1 wrote. That is an old value, not a missing
  one, and nothing reports it. Give the attribute a getter and share it.

Check both while you look through the project, because neither shows up as an error.
