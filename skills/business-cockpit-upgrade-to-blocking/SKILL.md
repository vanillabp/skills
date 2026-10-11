---
name: business-cockpit-upgrade-to-blocking
description: Upgrade a customized VanillaBP Business Cockpit application from the reactive container to the blocking one. Surveys a derived Spring Boot application depending on io.vanillabp.businesscockpit:container for everything the change from Spring WebFlux to Spring MVC on virtual threads touches, then applies it. Covers the non-reactive API server artifacts, SecurityFilterChain replacing SecurityWebFilterChain, the blocking JWT classes, the renamed public matchers of WebSecurityConfiguration, UserContext replacing ReactiveUserContext, MongoTemplate replacing ReactiveMongoTemplate, the WebMVC gateway proxy, SseEmitter instead of Flux<ServerSentEvent>, and the renamed spring.mvc and spring.cloud.gateway.server.webmvc properties. Use whenever an application depends on the business cockpit container, whenever Mono, Flux or ServerWebExchange appear in a business cockpit customization, and whenever a build against the blocking container fails on those types.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Upgrading a customized Business Cockpit to the blocking runtime

The Business Cockpit container was written in reactive style, on Spring WebFlux, Reactor and
the reactive MongoDB driver. It has been rewritten in classic blocking style, on Spring MVC
with virtual threads enabled. A customized business cockpit is a Spring Boot application of its
own. It depends on `io.vanillabp.businesscockpit:container` (from `0.8.0` on: on the new
library `io.vanillabp.businesscockpit:business-cockpit`, see
[references/dependencies.md](references/dependencies.md)). It extends
`io.vanillabp.cockpit.BusinessCockpitApplication` and overrides parts of it. So that rewrite
reaches into the derived application as well.

Everything which is not a signature survives the upgrade. The MongoDB data is untouched, and no
collection is renamed. The REST contracts keep their OpenAPI specifications. So a workflow
module which reports into the cockpit needs no change and no new deployment. The extension
points keep their bean names. So whatever the derived application overrides today, it still
overrides afterwards.

What changes is the type of nearly everything which crosses those extension points: no `Mono`,
no `Flux`, no `ServerWebExchange`, no `ReactiveMongoTemplate`, no `SecurityWebFilterChain`. One
set of public constants is renamed as well, the request matchers of `WebSecurityConfiguration`.
That one breaks a derived chain at compile time.

## Which versions this describes

The starting point is a customized application on any reactive version of the container and of
commons. That is `0.4.x` and everything before it. An application on `0.3.x` or earlier has a
second upgrade to do along with this one, from Spring Boot 3 to 4 and from Java 17 to 21.
[references/dependencies.md](references/dependencies.md) says where that one ends and this one
begins. An application already on `0.4.x` has that part behind it. Only the step from reactive
to blocking is left.

The target is `0.8.0`, the first release built on the blocking runtime. The versions between
`0.4.x` and `0.8.0` are kept for further releases of the reactive line. So a version in that
range is still a starting point of this upgrade, not a target. Where the release notes of the
`business-cockpit` repository and this skill disagree, the release notes win.

## How to run this

Look first, report, then change. The compiler finds most of the reactive code for you. But it
finds it one file at a time, and an upgrade done that way loses the overview of what is left.

### Step 0: survey the application

Work through [references/detection.md](references/detection.md). It lists what to grep for and
what each finding means. Write down a list of what you found before you touch a file:

- the container version in use and the Spring Boot version the application builds against,
- every dependency on a `-server-reactive` API artifact, on WebFlux and on the reactive gateway,
- every controller which implements a generated API interface or extends one of the container's
  abstract controllers,
- every security bean, and for each one whether it overrides a container bean by name,
- every reference to a `...WebExchangeMatcher` constant of `WebSecurityConfiguration`, which is
  renamed,
- every use of `ReactiveUserContext`, `ReactiveMongoTemplate`, `ReactiveMongoRepository`,
  `ReactiveChangeStreamUtils` and the reactive JWT classes,
- every changeset method and the type of its parameter,
- every reactive property in every YAML file, profiles and test resources included,
- every remaining `Mono`, `Flux`, `block()`, `subscribe()` and `Schedulers` call.

Show that list to the user, with the count per item. It is also the checklist you tick off.

### Step 1: swap the dependencies

Follow [references/dependencies.md](references/dependencies.md). The API artifacts generated
for Reactor are gone. The servlet starters replace the WebFlux starters, and the gateway
starter changes its transport. Do this first. Then from here on the compiler agrees with you
about what is still reactive.

### Step 2: convert the web layer

[references/web-layer.md](references/web-layer.md) covers controllers, the update stream and
the workflow module proxy. Take the signatures from the generated interfaces, not from that
file. The OpenAPI specification did not change, but the generator did. The interface on the
classpath decides what a controller has to implement.

### Step 3: convert the security configuration

[references/security.md](references/security.md) has the mapping of types and the bean names.
Read it even if the application only overrides one chain. The container's own chains move too,
and a derived chain works together with them through `@Order`. It also lists the renamed public
matchers of `WebSecurityConfiguration`. That is the one place where a derived application
breaks on a name, not on a type.

### Step 4: convert the data access

[references/data-access.md](references/data-access.md) covers repositories, templates,
changesets and change streams. Look carefully at changeset methods. They keep their identity in
the database, so a changeset which already ran must not run again. Rewriting its body is only
safe because nothing will call it any more.

### Step 5: move the configuration

Key by key, following [references/configuration.md](references/configuration.md). Search every
profile file and every test resource. A reactive property left behind is not reported. It is
simply not bound, so the setting goes back to its default without a word.

### Step 6: remove what Reactor left behind

Once the application compiles, the remaining `Mono` and `Flux` are the ones nothing forced you
to touch: bridges into blocking code, subscriptions nobody waits for, and test helpers. They
still compile as long as `reactor-core` is anywhere on the classpath, so nothing points at
them. [references/behavior-changes.md](references/behavior-changes.md) explains why each of
them behaves differently now.

### Step 7: verify by running it

Build the application, start it against a MongoDB with a replica set, and read the startup log
from the top. Then check the four things which the compiler cannot check:

1. Log in through the UI. This tests the JWT filter, the security chain and the user context in
   one request. The response to the sign-in request itself carries the JWT cookie. The client
   then asks for the current user a second time, with that cookie.
2. Watch the update stream. Open the network tab on `/gui/api/v1/updates`. Then complete a user
   task somewhere, and see the event arrive. If a stream opens and dies after a few seconds, a
   read timeout somewhere on the way caused it, not a broken listener.
3. Open a user task form served through the proxy. That path goes through the router function
   which `MicroserviceProxyRegistry` builds. That part has the least in common with its
   reactive predecessor.
4. Run the application's own tests. Read the ones you had to rewrite, not only their results.

## Two things that are easy to miss

Both are silent: the application starts, and most requests work.

### The security context is thread local again

Reactor carried the authenticated user in the subscriber context, which followed the work
wherever it was scheduled. `SecurityContextHolder` follows the thread instead. So any work
handed to an executor, an `@Async` method or a scheduled task starts without a user. Code which
read the current user deep inside such a chain now has to receive it as a parameter.
`PassiveJwtSecurityFilter` also clears the context in a `finally` block when the request ends.
So nothing lives on after it by accident.

The same change turns an unauthenticated call to `/gui/api/v1/app/current-user` into an HTTP
401. Under Reactor the missing security context made the chain empty, and the endpoint answered
200 with no body. See [references/behavior-changes.md](references/behavior-changes.md).

### A read timeout on the outbound HTTP client cuts the update stream

The proxy to the workflow modules is now a plain blocking HTTP client. A read timeout on it
applies to a streamed response just as to a short one. A proxied stream of server-sent events
is idle between events. So any read timeout tears it down in the middle, and the UI reconnects
in a loop. That is why the container sets `spring.http.clients.connect-timeout` and leaves
`spring.http.clients.read-timeout` unset, on purpose. If the derived application sets one, it
undoes that. The symptom appears only on a connection which is kept open, not on a single
request.

## Source

The `business-cockpit` repository itself is the description of this release which decides.
Above all
[container/README.md](https://github.com/vanillabp/business-cockpit/blob/main/container/README.md)
says how a customized application is built. Also read the release notes of the version you
upgrade to. Copy the patterns from the container's own `WebSecurityConfiguration`,
`LoginApiController` and `MicroserviceProxyRegistry`. `development/simulator` and
`development/dev-shell-simulator` in the same repository show the same JWT classes used from a
workflow module.

Upgrading the VanillaBP runtime itself is a different job with its own skill,
`vanillabp-upgrade-v1-to-v2`.
