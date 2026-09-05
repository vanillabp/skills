---
name: business-cockpit-upgrade-to-blocking
description: Upgrade a customized VanillaBP Business Cockpit application from the reactive container to the blocking one. Surveys a derived Spring Boot application depending on io.vanillabp.businesscockpit:container for everything the change from Spring WebFlux to Spring MVC on virtual threads touches, then applies it. Covers the non-reactive API server artifacts, SecurityFilterChain replacing SecurityWebFilterChain, the blocking JWT classes, the renamed public matchers of WebSecurityConfiguration, UserContext replacing ReactiveUserContext, MongoTemplate replacing ReactiveMongoTemplate, the WebMVC gateway proxy, SseEmitter instead of Flux<ServerSentEvent>, and the renamed spring.mvc and spring.cloud.gateway.server.webmvc properties. Use whenever an application depends on the business cockpit container, whenever Mono, Flux or ServerWebExchange appear in a business cockpit customization, and whenever a build against the blocking container fails on those types.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Upgrading a customized Business Cockpit to the blocking runtime

The Business Cockpit container was written in reactive style, on Spring WebFlux, Reactor and the
reactive MongoDB driver. It has been rewritten in classic blocking style, on Spring MVC with
virtual threads enabled. A customized business cockpit is an own Spring Boot application which
depends on `io.vanillabp.businesscockpit:container`, extends
`io.vanillabp.cockpit.BusinessCockpitApplication` and overrides parts of it, so that rewrite
reaches into the derived application as well.

What survives the upgrade is everything that is not a signature. The MongoDB data is untouched
and no collection is renamed. The REST contracts keep their OpenAPI specifications, so a
workflow module reporting into the cockpit needs no change and no redeployment. The extension
points keep their bean names, so whatever the derived application overrides today it still
overrides afterwards.

What changes is the type of nearly everything crossing those extension points: no `Mono`, no
`Flux`, no `ServerWebExchange`, no `ReactiveMongoTemplate`, no `SecurityWebFilterChain`. One set of
public constants is renamed as well, the request matchers of `WebSecurityConfiguration`, and that
one breaks a derived chain at compile time.

## Which versions this describes

The source is a customized application on any reactive container and commons version. That is
`0.4.x` and everything before it. An application on `0.3.x` or earlier carries a second upgrade
along with this one, from Spring Boot 3 to 4 and from Java 17 to 21;
[references/dependencies.md](references/dependencies.md) says where that one ends and this one
begins. An application already on `0.4.x` has that part behind it and only the reactive to
blocking step left.

The target is the first release built on the blocking runtime, which follows the `0.4.x` line. Its
number was not fixed when this skill was written, so look it up before you write it into a POM:
read the release notes of the `business-cockpit` repository and take the first version whose notes
describe the blocking runtime. Where the release notes and this skill disagree, the release notes
win.

## How to run this

Survey first, report, then change. The compiler finds most of the reactive code for you, but it
finds it one file at a time, and an upgrade applied that way loses the overview of what is left.

### Step 0: survey the application

Work through [references/detection.md](references/detection.md). It lists what to grep for and
what each finding means. Produce a written inventory before touching a file:

- the container version in use and the Spring Boot version the application builds against,
- every dependency on a `-server-reactive` API artifact, on WebFlux and on the reactive gateway,
- every controller implementing a generated API interface or extending one of the container's
  abstract controllers,
- every security bean, and for each one whether it overrides a container bean by name,
- every reference to a `...WebExchangeMatcher` constant of `WebSecurityConfiguration`, which is
  renamed,
- every use of `ReactiveUserContext`, `ReactiveMongoTemplate`, `ReactiveMongoRepository`,
  `ReactiveChangeStreamUtils` and the reactive JWT classes,
- every changeset method and the type of its parameter,
- every reactive property in every YAML file, profiles and test resources included,
- every remaining `Mono`, `Flux`, `block()`, `subscribe()` and `Schedulers` call.

Show that inventory to the user with the count per item. It is also the checklist you tick off.

### Step 1: swap the dependencies

Follow [references/dependencies.md](references/dependencies.md). The API artifacts generated for
Reactor are gone, the WebFlux starters give way to the servlet ones, and the gateway starter
changes its transport. Do this first, so that from here on the compiler agrees with you about
what is still reactive.

### Step 2: convert the web layer

Controllers, the update stream and the workflow module proxy are in
[references/web-layer.md](references/web-layer.md). Take the signatures from the generated
interfaces rather than from that file: the OpenAPI specification did not change, but the
generator did, and the interface on the classpath is the authority for what a controller has to
implement.

### Step 3: convert the security configuration

[references/security.md](references/security.md) has the type mapping and the bean names. Read it
even if the application only overrides one chain, because the container's own chains move too and
a derived chain interacts with them through `@Order`. It also lists the renamed public matchers of
`WebSecurityConfiguration`, which is the one place where a derived application breaks on a name
rather than on a type.

### Step 4: convert the data access

Repositories, templates, changesets and change streams are in
[references/data-access.md](references/data-access.md). Changeset methods deserve a careful look:
they keep their identity in the database, so a changeset which already ran must not run again,
and rewriting its body is safe only because it will not be invoked any more.

### Step 5: move the configuration

Key by key, following [references/configuration.md](references/configuration.md). Search every
profile file and every test resource. A reactive property left behind is not reported, it is
simply not bound, so the setting silently reverts to its default.

### Step 6: remove what Reactor left behind

Once the application compiles, the remaining `Mono` and `Flux` are the ones nothing forced you to
touch: bridges into blocking code, fire-and-forget subscriptions and test helpers. They still
compile as long as `reactor-core` is anywhere on the classpath, so nothing points at them.
[references/behavior-changes.md](references/behavior-changes.md) explains why each of them
behaves differently now.

### Step 7: verify by running it

Build the application, start it against a MongoDB with a replica set, and read the startup log
from the top. Then check the four things which the compiler cannot check:

1. Log in through the UI. This exercises the JWT filter, the security chain and the user context
   in one request. The response to the sign-in request itself carries the JWT cookie, and the
   client asks for the current user a second time with it.
2. Watch the update stream. Open the network tab on `/gui/api/v1/updates`, then complete a user
   task somewhere and see the event arrive. A stream that opens and dies after a few seconds is a
   read timeout somewhere on the way, not a broken listener.
3. Open a user task form served through the proxy. That path goes through the router function
   built by `MicroserviceProxyRegistry`, which is the part with the least in common with its
   reactive predecessor.
4. Run the application's own tests, and read the ones you had to rewrite rather than only their
   results.

## Two things that are easy to miss

Both are silent, in the sense that the application starts and most requests work.

### The security context is thread local again

Reactor carried the authenticated user in the subscriber context, which followed the work wherever
it was scheduled. `SecurityContextHolder` follows the thread instead, so any work handed to an
executor, a `@Async` method or a scheduled task starts without a user. Code which read the current
user deep inside such a chain has to receive it as a parameter now. `PassiveJwtSecurityFilter` also
clears the context in a `finally` block when the request ends, so nothing outlives it by accident.

The same change turns an unauthenticated call to `/gui/api/v1/app/current-user` into an HTTP 401.
Under Reactor the missing security context made the chain empty and the endpoint answered 200 with
no body. See [references/behavior-changes.md](references/behavior-changes.md).

### A read timeout on the outbound HTTP client cuts the update stream

The proxy to the workflow modules is a plain blocking HTTP client now, and a read timeout on it
applies to a streamed response just as it does to a short one. A proxied server-sent-event stream
is idle between events, so any read timeout tears it down mid-stream and the UI reconnects in a
loop. The container therefore sets `spring.http.clients.connect-timeout` and deliberately leaves
`spring.http.clients.read-timeout` unset. If the derived application sets one, it undoes that, and
the symptom appears only on a connection kept open, not on a single request.

## Source

The authoritative description of this release is the `business-cockpit` repository itself, in
particular [container/README.md](https://github.com/vanillabp/business-cockpit/blob/main/container/README.md)
for how a customized application is built, and the release notes of the version you upgrade to.
The container's own `WebSecurityConfiguration`, `LoginApiController` and `MicroserviceProxyRegistry`
are the patterns to copy from, and `development/simulator` and `development/dev-shell-simulator` in
the same repository show the same JWT classes used from a workflow module.

Upgrading the VanillaBP runtime itself is a different job with its own skill,
`vanillabp-upgrade-v1-to-v2`.
