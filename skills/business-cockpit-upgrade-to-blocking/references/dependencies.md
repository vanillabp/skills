# Dependencies

## Runtime

| | Container 0.3.x | Container 0.4.x | Container 0.8.0 (blocking) |
|---|---|---|---|
| Java | 17 | 21 | 21 |
| Spring Boot | 3.0.x | 4.1.x | 4.1.x |
| Spring Cloud | 2022.0.x | 2025.1.x | 2025.1.x |
| Web stack | WebFlux on Reactor Netty | WebFlux on Reactor Netty | Spring MVC on Tomcat |

The jump in Java and Spring Boot happened inside the reactive line, in `0.4.0`. So an
application which comes from `0.3.x` has two upgrades to do at once. The part from Spring Boot
3 to 4 is Spring's change, not the Business Cockpit's. `spring-boot-autoconfigure` was split
into modules. So imports of your own code move, autoconfiguration classes live in artifacts
named after their technology, and the test slices need explicit dependencies. Follow Spring
Boot's own migration guide for it, and do it as a separate, earlier commit. Everything else in
this skill assumes it is done.

An application already on `0.4.x` has that part behind it. Only the step from reactive to
blocking remains.

## The container itself

With `0.8.0` the former `container` module is split in two. The library `business-cockpit`
carries everything a derived application uses, and it cannot run on its own. The remaining
`container` is the reference application you can run: a main class, the concrete GUI
controllers, demo users of the `local` profile, and the application yamls. So a customized
business cockpit switches its dependency:

```xml
<dependency>
  <groupId>io.vanillabp.businesscockpit</groupId>
  <artifactId>business-cockpit</artifactId>
</dependency>
```

Java packages are unchanged, so imports stay as they are. The application class still extends
`io.vanillabp.cockpit.BusinessCockpitApplication`. That class now lives in the library, and no
longer carries `@SpringBootApplication` or a main method. The derived application's own class
provides both, as it already did. The thinner runtime has two results worth attention. First,
the concrete GUI controllers of the reference application are no longer on the classpath. So a
derived application implements its own subclasses of the abstract controllers. Most already do,
that is what the abstract classes are for. Second, the BPMS API must be configured (realm name
and credentials), or the application fails at startup.

## API artifacts

The API modules were published twice, once for the blocking Spring generator and once for the
reactive one. The reactive variants are removed.

| Before | After |
|---|---|
| `io.vanillabp.businesscockpit:official-gui-api-server-reactive` | `io.vanillabp.businesscockpit:official-gui-api-server` |
| `io.vanillabp.businesscockpit:bpms-api-server-reactive` | `io.vanillabp.businesscockpit:bpms-api-server` |
| `io.vanillabp.businesscockpit:workflow-provider-api-server` | unchanged, it never had a reactive variant |

Both blocking artifacts already existed in the reactive line, because `development/simulator`
and `development/dev-shell-simulator` compiled against them. The container switched to them by
removing the generator option `<reactive>true</reactive>` from its POM.

Package and type names of the generated models are the same in both variants. So imports of
model classes stay as they are. Only the API interfaces differ, as [web-layer.md](web-layer.md)
describes.

## Starters and Boot modules

| Before | After |
|---|---|
| `org.springframework.boot:spring-boot-starter-webflux` | `org.springframework.boot:spring-boot-starter-web` |
| `org.springframework.boot:spring-boot-webflux` | `org.springframework.boot:spring-boot-webmvc` |
| `org.springframework.boot:spring-boot-starter-data-mongodb-reactive` | `org.springframework.boot:spring-boot-starter-data-mongodb` |
| `org.springframework.cloud:spring-cloud-starter-gateway-server-webflux` (Gateway 5) | `org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc` |
| `org.springframework.cloud:spring-cloud-starter-gateway` (Gateway 4, so `0.3.x` and earlier) | `org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc` |
| `io.projectreactor:reactor-test` | nothing, delete it |

The WebMVC gateway starter pulls in `spring-boot-starter-web` itself. So an application which
declares the gateway does not need the web starter separately.

`commons` used to depend on both the reactive and the blocking MongoDB starter, because it held
both sets of classes side by side. Only the blocking one is left, and a derived application
inherits that.

## What has to leave the classpath

Reactor often survives an upgrade, because something still pulls it in. Then the leftover
`Mono` in a corner of the application keeps compiling. After you swap the dependencies, check
that neither `spring-webflux` nor `reactor-core` is on the compile classpath any more:

```bash
mvn dependency:tree -Dincludes=io.projectreactor:*,org.springframework:spring-webflux
```

Once the result is empty, the compiler starts finding the rest of the work for you. If
something really still needs Reactor, say which dependency and why. Do not leave it
unexplained.
