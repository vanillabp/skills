# Surveying a customized business cockpit

Run these from the project root before changing anything. Each row names what to look for and
what a hit means. Adapt the tool to what is available; the patterns matter, not the command.

## Runtime and dependencies

| Look for | Command | Meaning |
|---|---|---|
| Container version | `grep -rn "io.vanillabp.businesscockpit" -A2 --include=pom.xml --include="*.gradle*" .` | the version of `container` and `commons` the application inherits its backend from |
| Reactive API artifacts | `grep -rn -- "-server-reactive" --include=pom.xml .` | `official-gui-api-server-reactive` and `bpms-api-server-reactive` do not exist any more, see [dependencies.md](dependencies.md) |
| WebFlux | `grep -rn "webflux" --include=pom.xml .` | starter and Boot module both change |
| Reactive MongoDB | `grep -rn "data-mongodb-reactive" --include=pom.xml .` | replaced by the blocking starter |
| Gateway | `grep -rn "gateway" --include=pom.xml .` | the starter carries its transport in the artifact name |
| Reactor on the test path | `grep -rn "reactor-test" --include=pom.xml .` | drops out together with the tests using it |
| Java version | `grep -rn "maven.compiler\|version.java\|<release>" --include=pom.xml .` | has to be 21 |
| Spring Boot version | `grep -rn "spring-boot-dependencies\|spring-boot-starter-parent" -A2 --include=pom.xml .` | a Boot 3 application also has the Spring Boot 4 upgrade ahead of it |

## Web layer

| Look for | Command | Meaning |
|---|---|---|
| Reactive types | `grep -rn "Mono<\|Flux<" --include="*.java" .` | the headline number of the inventory. Count it before and after |
| Exchange parameter | `grep -rn "ServerWebExchange" --include="*.java" .` | the generated interfaces no longer pass one |
| Own controllers on generated interfaces | `grep -rn "implements .*Api\b" --include="*.java" .` | every signature is dictated by the regenerated interface |
| Controllers extending the container | `grep -rn "extends Abstract.*GuiApiController" --include="*.java" .` | the abstract hook methods lose their `Mono` too, so overrides stop overriding silently until the class fails to compile |
| Server-sent events | `grep -rn "ServerSentEvent\|TEXT_EVENT_STREAM" --include="*.java" .` | the update endpoint is written with `SseEmitter` now |
| Web configuration | `grep -rn "WebFluxConfigurer\|@EnableWebFlux\|WebFilter\|CorsWebFilter" --include="*.java" .` | each has a servlet counterpart, see [web-layer.md](web-layer.md) |
| Reactive codecs | `grep -rn "JacksonJsonEncoder\|JacksonJsonDecoder\|ServerCodecConfigurer\|CodecCustomizer" --include="*.java" .` | the message converters take over, and a `JsonMapperBuilderCustomizer` is all that is left to declare |
| Error handling | `grep -rn "ErrorWebExceptionHandler\|AbstractErrorWebExceptionHandler\|WebExceptionHandler" --include="*.java" .` | the single-page-app fallback is a `@ControllerAdvice` on `NoResourceFoundException` now |
| Login recording | `grep -rn "UserLoginUpsertWebFilter\|userLoginUpsertWebFilter" --include="*.java" .` | renamed to `UserLoginUpsertFilter`, and the bean method with it |
| Outbound calls | `grep -rn "WebClient" --include="*.java" .` | `WebClient` lives in `spring-webflux`, which leaves the classpath. `RestClient` is the replacement |
| Proxy registry | `grep -rn "MicroserviceProxyRegistry\|RefreshRoutesEvent\|RouteLocator" --include="*.java" .` | the registry keeps its methods but changes what it is; `RefreshRoutesEvent` is gone |

## Security

| Look for | Command | Meaning |
|---|---|---|
| Reactive chains | `grep -rn "SecurityWebFilterChain\|ServerHttpSecurity" --include="*.java" .` | become `SecurityFilterChain` and `HttpSecurity` |
| Overridden container beans | `grep -rn "guiHttpSecurity\|bpmsApiHttpSecurity\|userDetailsProvider\|jwtMapper" --include="*.java" .` | `guiHttpSecurity` and `userDetailsProvider` make the container back off on the bean name, so those names have to survive the conversion unchanged |
| Enabling annotations | `grep -rn "EnableWebFluxSecurity\|EnableReactiveMethodSecurity" --include="*.java" .` | become `@EnableWebSecurity` and `@EnableMethodSecurity` |
| Matchers | `grep -rn "ServerWebExchangeMatcher\|PathPatternParserServerWebExchangeMatcher" --include="*.java" .` | become `RequestMatcher` and `PathPatternRequestMatcher.pathPattern(...)` |
| The container's public matchers | `grep -rn "WebExchangeMatcher" --include="*.java" .` | `appInfoWebExchangeMatcher` and its four siblings on `WebSecurityConfiguration` are renamed to `...RequestMatcher`, see [security.md](security.md) |
| Reactive user details | `grep -rn "MapReactiveUserDetailsService\|ReactiveAuthenticationManager\|UserDetailsRepositoryReactiveAuthenticationManager" --include="*.java" .` | become `InMemoryUserDetailsManager` and a `ProviderManager` over a `DaoAuthenticationProvider` |
| Reactive JWT classes | `grep -rn "JwtSecurityWebFilter\|JwtServerSecurityContextRepository\|JwtBearerTokenConverter\|ReactiveJwtUserDetailsProvider\|ReactiveUserDetailsProvider" --include="*.java" .` | removed from `commons`, blocking twins take over except for `JwtBearerTokenConverter`, which has none |
| User context | `grep -rn "ReactiveUserContext\|usercontext.reactive\|getUserLoggedInAsMono\|getUserLoggedInDetailsAsMono" --include="*.java" .` | the whole `usercontext.reactive` package is gone; the blocking `UserContext` was the parent class all along |

## Data access

| Look for | Command | Meaning |
|---|---|---|
| Reactive repositories | `grep -rn "ReactiveMongoRepository\|ReactiveCrudRepository\|ReactiveSortingRepository" --include="*.java" .` | become `MongoRepository` or `ListCrudRepository` |
| Reactive template | `grep -rn "ReactiveMongoTemplate" --include="*.java" .` | becomes `MongoTemplate` |
| Changesets | `grep -rn "@Changeset" -A3 --include="*.java" .` | look at the parameter type of each method, that is what changes |
| Change streams | `grep -rn "ReactiveChangeStreamUtils" --include="*.java" .` | becomes `ChangeStreamUtils`, and the shape of a subscription changes with it |
| Lifecycle callbacks | `grep -rn "Reactive.*Callback\|ReactiveAuditorAware" --include="*.java" .` | drop the `Reactive` prefix and return the entity instead of a publisher |
| Blocking bridges | `grep -rn "\.block()\|\.blockFirst()\|\.blockLast()\|\.toStream()" --include="*.java" .` | usually deletable together with the reactive call they were unwrapping |

## Configuration

| Look for | Command | Meaning |
|---|---|---|
| Static resources | `grep -rn "webflux" --include="*.yaml" --include="*.yml" --include="*.properties" .` | `spring.webflux.static-path-pattern` becomes `spring.mvc.static-path-pattern` |
| Gateway | `grep -rn "spring.cloud.gateway\|cloud:" -A5 --include="*.yaml" --include="*.yml" --include="*.properties" .` | moves under `spring.cloud.gateway.server.webmvc` |
| Virtual threads | `grep -rn "threads.virtual\|virtual:" --include="*.yaml" --include="*.yml" --include="*.properties" .` | the container enables it; make sure no profile of the application switches it off |
| Read timeouts | `grep -rn "read-timeout\|readTimeout" --include="*.yaml" --include="*.yml" --include="*.properties" .` | `spring.http.clients.read-timeout` cuts proxied event streams, see [configuration.md](configuration.md) |
| Netty and server settings | `grep -rn "server.netty\|reactor.netty" --include="*.yaml" --include="*.yml" --include="*.properties" .` | Tomcat serves the application now, so these do nothing |

Search profile files and test resources too. Configuration is where an upgrade is most often left
half done, and an unbound key produces no message.

## Tests

| Look for | Command | Meaning |
|---|---|---|
| Reactive test clients | `grep -rn "WebTestClient\|MockServerWebExchange\|StepVerifier" --include="*.java" .` | `MockMvcTester` and plain assertions replace them |
| Reactive slices | `grep -rn "@WebFluxTest\|@DataMongoTest" --include="*.java" .` | `@WebMvcTest`, and the Mongo slice now boots the blocking template |

## What no grep will find

Say these out loud in the report, because they need a person to answer:

- Work handed to an executor, a `@Async` method or a scheduled task which reads the current user.
  It compiled and worked while Reactor propagated the context, and it returns no user now.
- A `subscribe()` whose exceptions nobody ever saw. Once it is a plain method call, they reach
  the caller, which is an improvement that shows up as a new failure.
- A `synchronized` block around a blocking call. On virtual threads that pins the carrier thread
  for the duration of the call.
- Any read timeout on the way to a workflow module, whether from
  `spring.http.clients.read-timeout` or from a reverse proxy in front of the cockpit. The container
  pings every 27 seconds, so a shorter timeout cuts the update stream and no build fails.
- What the Kafka consumer should do with a record whose handling now throws instead of being
  swallowed. Retries and a dead letter topic are a decision, see
  [behavior-changes.md](behavior-changes.md).
- Anything reading the empty HTTP 200 from `/gui/api/v1/app/current-user` as "not logged in". It
  answers 401 now.
