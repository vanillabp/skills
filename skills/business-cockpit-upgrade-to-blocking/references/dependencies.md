# Dependencies

## Runtime

| | Container 0.3.x | Container 0.4.x | The first blocking release |
|---|---|---|---|
| Java | 17 | 21 | 21 |
| Spring Boot | 3.0.x | 4.1.x | 4.1.x |
| Spring Cloud | 2022.0.x | 2025.1.x | 2025.1.x |
| Web stack | WebFlux on Reactor Netty | WebFlux on Reactor Netty | Spring MVC on Tomcat |

The Java and Spring Boot jump happened inside the reactive line, in `0.4.0`. An application coming
from `0.3.x` therefore carries two upgrades at once. The Spring Boot 3 to 4 part is Spring's change
and not the Business Cockpit's: `spring-boot-autoconfigure` was modularized, so imports of your own
code move, autoconfiguration classes live in artifacts named after their technology, and the test
slices need explicit dependencies. Follow Spring Boot's own migration guide for it and treat it as
a separate, earlier commit. Everything else in this skill assumes it is done.

An application already on `0.4.x` has that part behind it, and only the reactive to blocking part
remains.

## The container itself

The coordinates do not change. A customized business cockpit declares

```xml
<dependency>
  <groupId>io.vanillabp.businesscockpit</groupId>
  <artifactId>container</artifactId>
</dependency>
```

and extends `io.vanillabp.cockpit.BusinessCockpitApplication`, before as afterwards. Only the
version moves.

## API artifacts

The API modules were published twice, once for the blocking Spring generator and once for the
reactive one. The reactive variants are removed.

| Before | After |
|---|---|
| `io.vanillabp.businesscockpit:official-gui-api-server-reactive` | `io.vanillabp.businesscockpit:official-gui-api-server` |
| `io.vanillabp.businesscockpit:bpms-api-server-reactive` | `io.vanillabp.businesscockpit:bpms-api-server` |
| `io.vanillabp.businesscockpit:workflow-provider-api-server` | unchanged, it never had a reactive variant |

Both blocking artifacts existed in the reactive line already, because `development/simulator` and
`development/dev-shell-simulator` compiled against them. The container switched to them by dropping
the `<reactive>true</reactive>` generator option from its POM.

Package and type names of the generated models are identical in both variants, so imports of
model classes stay as they are. Only the API interfaces differ, in the way described in
[web-layer.md](web-layer.md).

## Starters and Boot modules

| Before | After |
|---|---|
| `org.springframework.boot:spring-boot-starter-webflux` | `org.springframework.boot:spring-boot-starter-web` |
| `org.springframework.boot:spring-boot-webflux` | `org.springframework.boot:spring-boot-webmvc` |
| `org.springframework.boot:spring-boot-starter-data-mongodb-reactive` | `org.springframework.boot:spring-boot-starter-data-mongodb` |
| `org.springframework.cloud:spring-cloud-starter-gateway-server-webflux` (Gateway 5) | `org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc` |
| `org.springframework.cloud:spring-cloud-starter-gateway` (Gateway 4, so `0.3.x` and earlier) | `org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc` |
| `io.projectreactor:reactor-test` | nothing, delete it |

The WebMVC gateway starter pulls `spring-boot-starter-web` in itself, so an application declaring
the gateway does not need the web starter separately.

`commons` used to depend on both the reactive and the blocking MongoDB starter, because it held
both sets of classes side by side. Only the blocking one is left, which is what a derived
application inherits.

## What has to leave the classpath

Reactor tends to survive an upgrade because something still pulls it in, and then the leftover
`Mono` in a corner of the application keeps compiling. After the dependency swap, check that
neither `spring-webflux` nor `reactor-core` is on the compile classpath any more:

```bash
mvn dependency:tree -Dincludes=io.projectreactor:*,org.springframework:spring-webflux
```

An empty result is the point at which the compiler starts finding the rest of the work for you.
If something legitimately still needs Reactor, say which dependency and why, rather than leaving
it unexplained.
