# Behavior changes

These are the differences that survive a clean compile. Read them with the user, because several
need a decision rather than an edit.

## Exceptions from fire-and-forget calls now reach the caller

A `subscribe()` without a subscriber, or with one that only logs, swallowed everything the
publisher failed with. The same code as a plain method call throws into the caller.

```java
// before: the update is attempted, and a failure ends in the default error consumer
repository.save(entity).subscribe();

// after: a failure ends up wherever the caller handles it, or does not handle it
repository.save(entity);
```

This is an improvement, and it shows up as new failures in paths that looked healthy. Before
adding a `try`/`catch` around such a call, look at what the failure means. If the write was
optional, catching and logging is honest. If it was not optional and nobody noticed until now,
the fix belongs upstream.

Search the survey list of `subscribe(` hits and decide each one explicitly.

The container's own Kafka ingestion is the largest instance of this. Every `@KafkaListener` method
ended in `.subscribe()`, so a record which failed to be stored was logged by Reactor and the offset
was committed anyway. The handlers are plain calls now, and only `InvalidProtocolBufferException`
is caught, so anything else propagates out of the listener method into the Kafka listener
container. What happens next is the container's error handler and the consumer configuration, which
means retries and a dead letter topic are now a decision to make rather than something the code
quietly took away. A derived application which configured neither sees redelivery of a record whose
handling keeps failing.

## The security context follows the thread

Reactor carried the authenticated user in the subscriber context, so it reached every operator of
a chain no matter which scheduler ran it. `SecurityContextHolder` is thread local. Anything not
running on the request thread starts without a user:

- a `@Async` method,
- a `@Scheduled` method,
- work submitted to an executor,
- a MongoDB change stream listener, which runs on the listener container's pool,
- the Kafka listeners of the container.

Code which read the current user in such a place returned it before and returns nothing now.
Pass the user in as a parameter instead of reaching for the context. Where a value has to be
stamped on a write outside a request, `UpdateInformationAware.SYSTEM_USER` is what the container
uses.

`PassiveJwtSecurityFilter` clears the context in a `finally` block when the request completes, so
nothing leaks into the next request served by the same thread.

## An unauthenticated request for the current user answers 401

`/gui/api/v1/app/current-user` is one of the few paths the GUI chain permits without
authentication, so an unauthenticated request reaches the controller and asks `UserContext` for the
caller. `UserContext` throws `BcUnauthorizedException` when there is no security context, and
`RestfulExceptionHandler` turns that into an HTTP 401.

Under Reactor the same code answered 200 with an empty body, because
`ReactiveSecurityContextHolder.getContext()` completed empty when there was no context and the
chain never reached the `throw`. Anything relying on the empty 200 to decide "not logged in" has to
treat 401 as that answer now.

One case still answers an empty 200, and on purpose: the sign-in request itself. It authenticates
by basic auth, and it is its own response which carries the JWT cookie, so at the time the
controller runs there is no cookie to read the user details from. The client asks a second time
with the cookie.

## An update for an unknown user task or workflow now creates it

The BPMS API v1.1 endpoints `POST /bpms/api/v1_1/usertask/{userTaskId}/updated` and
`POST /bpms/api/v1_1/workflow/{workflowId}/updated` were
always meant to fall back to creating the entity when the cockpit has never seen it, so that a
cockpit added to a running system does not stay blind to what existed before. The reactive
implementation expressed that with `switchIfEmpty`, which re-subscribed to the request body after
`zipWith` had already consumed it, and the request ended in an HTTP 500. The blocking
implementation reads the body once into an object and branches on `null`, so the fallback works.

The same fallback exists on the Kafka path and had the same shape, without the request body
problem.

For a workflow module reporting into the cockpit this is a bug fix rather than a contract change.
For a derived application it means documents can appear where an update previously failed, so a
count of user tasks may go up after the upgrade.

## Ordering inside a request is now the order of the statements

A reactive chain interleaved work; a blocking method does one thing after another. Two places
where that is visible:

Latency is now additive. Two calls which ran concurrently through `Mono.zip` run one after the
other. If both are slow, say so and consider a structured concurrency construct rather than
leaving it as an accidental regression.

A read after a write sees the write. Under the reactive template it was possible to build a chain
where the two were not ordered relative to each other, and a guard written against that is now
dead code.

## Virtual threads and `synchronized`

Blocking is the point of this runtime, so a blocking call in a request needs no special
treatment. One construction does: a `synchronized` block around a blocking call pins the carrier
thread for the duration of the call on Java 21. Under load that turns a pool of virtual threads
back into a pool of platform threads.

Replace `synchronized` around I/O with a `ReentrantLock`, or move the I/O out of the guarded
section. Short `synchronized` blocks guarding only in-memory state are fine and need no change.

## Change streams deliver on their own threads

`ReactiveChangeStreamUtils` produced a `Flux` whose subscriber decided where the work ran.
`ChangeStreamUtils` registers a listener with a `MessageListenerContainer` which has its own thread
pool. So a listener starts without a security context, and a slow listener holds a thread of that
pool where it used to apply backpressure to a publisher. An exception escaping a listener is caught
and logged by the utility, which wraps every listener for exactly that, so the stream survives it.

The requirement on the database is unchanged. Change streams need a replica set, which is what the
Business Cockpit has always required.

## The update stream holds a thread

Each open server-sent-events connection occupies a request thread for as long as the browser keeps
it open. With `spring.threads.virtual.enabled: true` that is cheap. Without it, the number of
concurrent users is capped by the Tomcat thread pool, and the symptom is an application that stops
answering rather than one that answers slowly.

The container pings every subscriber every 27 seconds, and once more 300 milliseconds after
subscribing so the client sees the subscription stand. Every timeout between the browser and the
cockpit has to be longer than that interval, the outbound HTTP client of the proxy included, which
is why the container sets no read timeout on it at all. See
[configuration.md](configuration.md).

## A MongoDB problem still fails the boot, more legibly

The changesets run in a `@PostConstruct` against the blocking template, so a MongoDB which is not
reachable or not a replica set fails the boot at that point with the error from the driver. The
reactive version failed there too, through its `.block()` calls, but the stack trace is readable
now and names the changeset.

## Tests

`WebTestClient` and `StepVerifier` have no place any more. `MockMvcTester` covers the controller
tests, and a test which built a reactive chain and verified it step by step usually shrinks to a
call and an assertion. `@WebFluxTest` becomes `@WebMvcTest`.

Security chain tests are worth keeping in some form. The chains of the container and of a derived
application can be built and inspected without starting the full context, which matters because
that context needs MongoDB. The container's `WebSecurityChainTest` builds an `HttpSecurity` by hand
and asserts which chain takes which request and where the JWT filter sits inside the filter list,
because none of that fails loudly when it breaks.

The container also gained a black-box suite under `io.vanillabp.cockpit.itest` which boots the
application against a MongoDB replica set and a Kafka broker in Testcontainers and talks to it over
plain HTTP. That is the level at which the behavior changes on this page were pinned down, and a
derived application with customizations of its own is worth testing the same way.
