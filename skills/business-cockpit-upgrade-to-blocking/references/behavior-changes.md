# Behavior changes

These are the differences which remain after a clean compile. Read them with the user, because
several need a decision, not an edit.

## Exceptions from fire-and-forget calls now reach the caller

A `subscribe()` without a subscriber, or with one which only logs, swallowed every failure of
the publisher. The same code as a plain method call throws into the caller.

```java
// before: the update is attempted, and a failure ends in the default error consumer
repository.save(entity).subscribe();

// after: a failure ends up wherever the caller handles it, or does not handle it
repository.save(entity);
```

This is an improvement, and it shows up as new failures in paths which looked healthy. Before
you add a `try`/`catch` around such a call, look at what the failure means. If the write was
optional, catching and logging is honest. If it was not optional and nobody noticed until now,
the fix belongs further up.

Go through the `subscribe(` hits in your list, and decide each one explicitly.

The biggest case of this is the container's own intake from Kafka. Every `@KafkaListener`
method ended in `.subscribe()`. So when a record could not be stored, Reactor logged it, and
the offset was committed anyway. The handlers are plain calls now, and only
`InvalidProtocolBufferException` is caught. Anything else goes out of the listener method into
the Kafka listener container. What happens next depends on the container's error handler and
the consumer configuration. So retries and a dead letter topic are now a decision to make.
Before, the code quietly took that decision away. A derived application which configured
neither sees a record delivered again and again while its handling keeps failing.

## The security context follows the thread

Reactor carried the authenticated user in the subscriber context. So it reached every operator
of a chain, whichever scheduler ran it. `SecurityContextHolder` is thread local. Anything which
does not run on the request thread starts without a user:

- a `@Async` method,
- a `@Scheduled` method,
- work submitted to an executor,
- a MongoDB change stream listener, which runs on the listener container's pool,
- the Kafka listeners of the container.

Code which read the current user in such a place returned it before, and returns nothing now.
Pass the user in as a parameter, instead of reaching for the context. Where a write outside a
request needs a user stamped on it, the container uses `UpdateInformationAware.SYSTEM_USER`.

`PassiveJwtSecurityFilter` clears the context in a `finally` block when the request completes.
So nothing leaks into the next request which the same thread serves.

## An unauthenticated request for the current user answers 401

`/gui/api/v1/app/current-user` is one of the few paths the GUI chain permits without
authentication. So a request without authentication reaches the controller and asks
`UserContext` for the caller. `UserContext` throws `BcUnauthorizedException` when there is no
security context. `RestfulExceptionHandler` turns that into an HTTP 401.

Under Reactor the same code answered 200 with an empty body.
`ReactiveSecurityContextHolder.getContext()` completed empty when there was no context, and the
chain never reached the `throw`. Anything which relies on the empty 200 to decide "not logged
in" now has to treat 401 as that answer.

One case still answers an empty 200, on purpose: the sign-in request itself. It authenticates
by basic auth, and its own response carries the JWT cookie. So when the controller runs, there
is no cookie yet to read the user details from. The client asks a second time, with the cookie.

## An update for an unknown user task or workflow now creates it

The BPMS API v1.1 endpoints `POST /bpms/api/v1_1/usertask/{userTaskId}/updated` and
`POST /bpms/api/v1_1/workflow/{workflowId}/updated` were always meant to create the entity when
the cockpit has never seen it. So a cockpit added to a running system does not stay blind to
what existed before. The reactive code did that with `switchIfEmpty`. It subscribed to the
request body again after `zipWith` had already consumed it, and the request ended in an HTTP
500. The blocking code reads the body once into an object and branches on `null`, so the
fallback works.

The same fallback exists on the Kafka path and had the same shape, without the request body
problem.

For a workflow module which reports into the cockpit, this is a bug fix, not a change of
contract. For a derived application it means that documents can appear where an update failed
before. So a count of user tasks may go up after the upgrade.

## Ordering inside a request is now the order of the statements

A reactive chain mixed its steps. A blocking method does one thing after another. Two places
show that:

Waiting times now add up. Two calls which ran at the same time through `Mono.zip` now run one
after the other. If both are slow, say so. Consider a construct for structured concurrency,
instead of leaving it as an accidental step back.

A read after a write sees the write. Under the reactive template you could build a chain where
the two had no fixed order. A guard written against that is now dead code.

## Virtual threads and `synchronized`

Blocking is the point of this runtime. So a blocking call in a request needs no special
treatment. One construction does. On Java 21 a `synchronized` block around a blocking call pins
the carrier thread for as long as the call takes. Under load that turns a pool of virtual
threads back into a pool of platform threads.

Replace `synchronized` around I/O with a `ReentrantLock`, or move the I/O out of the guarded
section. Short `synchronized` blocks which only guard state in memory are fine and need no
change.

## Change streams deliver on their own threads

`ReactiveChangeStreamUtils` produced a `Flux` whose subscriber decided where the work ran.
`ChangeStreamUtils` registers a listener with a `MessageListenerContainer`, which has its own
thread pool. So a listener starts without a security context. A slow listener holds a thread of
that pool, where it used to slow down a publisher. If an exception escapes a listener, the
utility catches and logs it. It wraps every listener for exactly that, so the stream survives.

The requirement on the database is unchanged. Change streams need a replica set, and the
Business Cockpit has always required one.

Shutdown behaves differently as well. The blocking driver cannot interrupt a running poll of a
change stream. So stopping the application waits until the server answers the poll. The
property `mongodb.change-stream-max-await-time` limits that wait, and its default is one
second. A derived application on a database which bills polls, such as Azure Cosmos DB, can
raise it, and accepts a slower shutdown in return.

## The update stream holds a thread

Each open connection for server-sent events takes a request thread for as long as the browser
keeps it open. With `spring.threads.virtual.enabled: true` that is cheap. Without it, the
Tomcat thread pool limits the number of users at the same time. The symptom is an application
which stops answering, not one which answers slowly.

The container pings every subscriber every 27 seconds, and once more 300 milliseconds after
subscribing, so the client sees that the subscription stands. Every timeout between the browser
and the cockpit has to be longer than that interval, the outbound HTTP client of the proxy
included. That is why the container sets no read timeout on it at all. See
[configuration.md](configuration.md).

## A MongoDB problem still fails the boot, more legibly

The changesets run in a `@PostConstruct` against the blocking template. So a MongoDB which
cannot be reached, or is not a replica set, fails the boot at that point, with the error from
the driver. The reactive version failed there too, through its `.block()` calls. But now the
stack trace is readable and names the changeset.

## Tests

`WebTestClient` and `StepVerifier` have no place any more. `MockMvcTester` covers the
controller tests. A test which built a reactive chain and checked it step by step usually
shrinks to a call and an assertion. `@WebFluxTest` becomes `@WebMvcTest`.

Tests of the security chains are worth keeping in some form. You can build and inspect the
chains of the container and of a derived application without starting the full context. That
matters, because that context needs MongoDB. The container's `WebSecurityChainTest` builds an
`HttpSecurity` by hand. It asserts which chain takes which request, and where the JWT filter
sits inside the filter list. None of that fails loudly when it breaks.

The container also gained a black-box suite under `io.vanillabp.cockpit.itest`. It boots the
application against a MongoDB replica set and a Kafka broker in Testcontainers, and talks to it
over plain HTTP. The behavior changes on this page were pinned down at that level. A derived
application with customizations of its own is worth testing the same way.
