# Configuration

Every key below is a key of the derived application's own `application.yaml` and its profile
files. The business cockpit's own properties under `business-cockpit`, the MongoDB connection
under `spring.data.mongodb`, and the BPMS API credentials under `bpms-api` stay untouched.

A key which no longer exists is not reported. Spring binds what it recognizes and ignores the
rest. So a reactive key left behind is a setting which has no effect. Grep for the old names
instead of trusting a successful start.

## Static resources

| Before | After |
|---|---|
| `spring.webflux.static-path-pattern` | `spring.mvc.static-path-pattern` |

The container makes that move in its own `config/application.yaml`, keeping the value `"/**"`.

`spring.web.resources.static-locations` is common to both and does not move. A customized cockpit
serving its own webapp usually sets both keys together, so the pair is easy to spot.

## Virtual threads

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

New, and off in Spring Boot's own defaults. The container ships it in `config/application.yaml`
inside the `container` artifact, so a derived application does not have to declare it. Check
the running application instead of assuming, because a profile of the application can still set
it back to `false`.

Without virtual threads the application runs on the platform thread pool of Tomcat. There every
open update stream takes one of a few hundred threads. It works until the number of logged-in
users comes close to that limit, and then it stops accepting requests. So do not leave this
setting for later.

## Gateway

The proxy to the workflow modules moves from the reactive gateway to the WebMVC one, and its
properties move with it.

| Before | After |
|---|---|
| `spring.cloud.gateway.*` (Gateway 4 and earlier) | `spring.cloud.gateway.server.webmvc.*` |
| `spring.cloud.gateway.server.webflux.*` (Gateway 5) | `spring.cloud.gateway.server.webmvc.*` |

Two keys under that prefix matter for a business cockpit:

```yaml
spring:
  cloud:
    gateway:
      server:
        webmvc:
          streaming-media-types: [ "text/event-stream" ]
          streaming-buffer-size: 16384
```

The gateway copies a response chunk by chunk. For the media types listed in
`streaming-media-types` it flushes after every chunk. The default of that list already contains
`text/event-stream`. Everything else is copied without that flush per chunk. Set the list
explicitly only if a workflow module streams something under a different media type. Remember
that setting it replaces the default and does not add to it.

Two prefixes look right and are not. `spring.cloud.gateway.mvc.*` is the name from earlier
gateway versions. `spring.cloud.gateway.proxy-exchange.*` belongs to a different module
altogether.

## Outbound HTTP client

Timeouts for the proxy are no longer configured under the gateway prefix. They are Spring
Boot's global settings for HTTP clients, and they apply to every blocking HTTP client the
application builds. This is what the container sets:

```yaml
spring:
  http:
    clients:
      connect-timeout: 10s
```

There is no read timeout. Leaving it unset is a decision, not something forgotten. A read
timeout applies to a streamed response as well as to a short one. A stream of server-sent
events proxied from a workflow module is idle between events. So any read timeout would tear
such a stream down in the middle, and the UI would reconnect in a loop. A comment in the
container's `application.yaml` says so.

If a derived application sets `spring.http.clients.read-timeout`, that overrides the decision
for every outbound call the application makes, proxied streams included. The WebMVC gateway has
no timeouts per route. If you have to limit a slow workflow module, limit it somewhere which
can tell a stream from a request.

`spring.http.clients.imperative.factory` selects the request factory. Leave it at the default
unless there is a reason. The default is the JDK HTTP client, which parks on virtual threads
without pinning a carrier thread.

## What stops working

| Key | Why |
|---|---|
| `server.netty.*` | Tomcat serves the application now |
| `spring.reactor.*` | no Reactor on the classpath |

Settings for the whole server which do not depend on the transport are unchanged,
`server.port`, `server.compression.*` and `server.error.*` among them.

Also look for anything else under a prefix which only WebFlux has, in the application's own
YAML. The container has none. So this list only covers what a derived application probably
carries. An unbound key produces no message either way.
