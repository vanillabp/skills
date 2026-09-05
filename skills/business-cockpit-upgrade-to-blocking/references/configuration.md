# Configuration

Every key below is a key of the derived application's own `application.yaml` and its profile
files. The business cockpit's own properties under `business-cockpit`, the MongoDB connection
under `spring.data.mongodb` and the BPMS API credentials under `bpms-api` are untouched.

A key which no longer exists is not reported. Spring binds what it recognizes and ignores the
rest, so a reactive key left behind reads as a setting that has no effect. Grep for the old names
rather than trusting a successful start.

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

New, and off in Spring Boot's own defaults. The container ships it in
`config/application.yaml` inside the `container` artifact, so a derived application does not have
to declare it. Check the running application rather than assuming, because a profile of the
application can still set it back to `false`.

Without virtual threads the application runs on the platform thread pool of Tomcat, where every
open update stream occupies one of a few hundred threads. It works until the number of logged-in
users approaches that limit, and then stops accepting requests, so this is not a setting to leave
for later.

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

The gateway copies a response chunk by chunk and flushes per chunk for the media types listed in
`streaming-media-types`, whose default already contains `text/event-stream`. Everything else is
copied without that per-chunk flush. Set the list explicitly only if a workflow module streams
something under a different media type, and remember that setting it replaces the default rather
than adding to it.

Two prefixes that look right and are not: `spring.cloud.gateway.mvc.*` is the name from earlier
gateway versions, and `spring.cloud.gateway.proxy-exchange.*` belongs to a different module
altogether.

## Outbound HTTP client

Timeouts for the proxy are no longer configured under the gateway prefix. They are Spring Boot's
global HTTP client settings, and they apply to every blocking HTTP client the application builds.
This is what the container sets:

```yaml
spring:
  http:
    clients:
      connect-timeout: 10s
```

There is no read timeout, and leaving it unset is the decision, not an omission. A read timeout
applies to a streamed response as well as to a short one, and a server-sent-event stream proxied
from a workflow module is idle between events. Any read timeout would therefore tear such a stream
down mid-stream, and the UI would reconnect in a loop. The container's `application.yaml` carries a
comment saying so.

Setting `spring.http.clients.read-timeout` in a derived application overrides that decision for
every outbound call the application makes, proxied streams included, because per-route timeouts do
not exist in the WebMVC gateway. If a slow workflow module has to be bounded, bound it somewhere
that can tell a stream from a request.

`spring.http.clients.imperative.factory` selects the request factory. Leave it at the default
unless there is a reason: the default is the JDK HTTP client, which parks on virtual threads
without pinning a carrier thread.

## What stops working

| Key | Why |
|---|---|
| `server.netty.*` | Tomcat serves the application now |
| `spring.reactor.*` | no Reactor on the classpath |

Server-wide settings which are not transport specific, `server.port`, `server.compression.*` and
`server.error.*` among them, are unchanged.

Look for anything else under a WebFlux-only prefix in the application's own YAML as well. The
container has none, so this list only covers what a derived application is likely to carry, and an
unbound key produces no message either way.
