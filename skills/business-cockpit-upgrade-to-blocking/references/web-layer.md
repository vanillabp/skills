# The web layer

## Generated API interfaces

The OpenAPI specifications are unchanged. What changes is the generator: the `reactive` option is
off, so the emitted interfaces are plain Spring MVC.

Three things happen to every method:

1. The return type loses its wrapper. `Mono<ResponseEntity<X>>` becomes `ResponseEntity<X>`, and
   `Flux<X>` becomes `List<X>`.
2. A request body loses its wrapper too. `Mono<UserTasksRequest>` becomes `UserTasksRequest`.
3. The trailing `ServerWebExchange` parameter is gone.

A controller written against the reactive interface therefore turns from a chain into a body that
reads top to bottom:

```java
// before
@Override
public Mono<ResponseEntity<User>> currentUser(
        final String xRefreshToken,
        final ServerWebExchange exchange) {

    return userContext
            .getUserLoggedInDetailsAsMono()
            .map(user -> toApi(user))
            .map(ResponseEntity::ok);

}

// after
@Override
public ResponseEntity<User> currentUser(
        final String xRefreshToken) {

    final var user = userContext.getUserLoggedInDetails();
    return ResponseEntity.ok(toApi(user));

}
```

Do not copy signatures out of this file. Regenerate or resolve the API artifact first and let the
compiler tell you what each interface method looks like, because parameter order and nullability
follow the specification of the version you build against.

`Mono.zip` of the user context and the request body is the most frequent construction to unwind.
It becomes two statements, and the `Tuple2` accessors `getT1()` and `getT2()` disappear with it.

## Controllers extending the container

The container ships abstract controllers which a customized application extends to plug in its own
queries, among them `AbstractUserTaskListGuiApiController`,
`AbstractWorkflowListGuiApiController` and `AbstractWorkflowModulesGuiApiController`. Their
abstract hook methods carry the same conversion:

```java
// before
protected abstract Mono<Page<UserTask>> getUserTasks(
        UserDetails currentUser, int pageNumber, int pageSize, ...);

// after
protected abstract Page<UserTask> getUserTasks(
        UserDetails currentUser, int pageNumber, int pageSize, ...);
```

An override which keeps the old signature no longer overrides anything. In the abstract case the
class fails to compile, which is what you want. Where the hook was not abstract, add `@Override`
to every implementation of it before starting, so that the same mistake is also caught there.

The injected fields of those base classes change type as well: `ReactiveUserContext` becomes
`UserContext`, described in [security.md](security.md).

## The update stream

`/gui/api/v1/updates` returns an `SseEmitter` instead of a `Flux<ServerSentEvent<?>>`, and
`LoginApiController` holds one `UpdateEmitter` per subscribed browser in a map.

The event pipeline in front of it is unchanged. An application publishing its own push events keeps
using `ApplicationEventPublisher` with `io.vanillabp.cockpit.gui.api.v1.GuiEvent`, and the
container still collects those events through an `@EventListener(classes = GuiEvent.class)` and
flushes them to every emitter on a scheduled interval. A derived application which only publishes
`GuiEvent` therefore has nothing to do here. One which replaced the endpoint itself has to rewrite
it, and `LoginApiController` is the reference.

`SseEmitter` behaves differently from a `Flux` in ways that only show up while the application
runs. Each open stream holds a request thread for as long as the browser keeps the tab open, which
is affordable because the threads are virtual and is the reason `spring.threads.virtual.enabled` is
not optional. The emitter is created with a timeout of `Long.MAX_VALUE`, because the default
asynchronous request timeout would otherwise close a healthy stream.

The container pings every subscriber every 27 seconds so an idle stream stays open, plus once 300
milliseconds after subscribing so the client sees that the subscription stands. Every timeout
between the browser and the cockpit has to be longer than that interval, see
[behavior-changes.md](behavior-changes.md).

## Deep links into the single-page application

A path like `/tasklist/4711` matches no controller and no static resource, and the application
shell has to be returned instead of an error. That used to be an `ErrorWebExceptionHandler`
extending `AbstractErrorWebExceptionHandler`. `SpaNoHandlerFoundExceptionHandler` is a
`@ControllerAdvice` at `Ordered.HIGHEST_PRECEDENCE` now, with one `@ExceptionHandler` for
`NoResourceFoundException` and `NoHandlerFoundException` which answers `200` with the shell and
`Cache-Control: max-age=0`.

The precedence is what makes it work: the catch-all of `RestfulExceptionHandler` in `commons` would
otherwise turn every unknown path into an HTTP 500. A derived application with a `@ControllerAdvice`
of its own has to stay out of the way of both.

The reactive handler also rendered every other error status as JSON, because it sat in the error
path. The blocking one only answers the two not-found exceptions, and everything else goes through
the ordinary exception handling.

## JSON

`JsonConfiguration` in the container is down to its `JsonMapperBuilderCustomizer` beans. The
reactive codec beans it also declared, `JacksonJsonEncoder`, `JacksonJsonDecoder` and the
`WebFluxConfigurer` registering them, are gone: Spring Boot builds the `JsonMapper` from the
customizers and hands it to the message converter Spring MVC uses. The wire format is unchanged.

A derived application which registered codecs of its own drops them the same way and keeps only its
customizers.

## The workflow module proxy

`MicroserviceProxyRegistry` keeps its public API:

| | |
|---|---|
| `WORKFLOW_MODULES_PATH_PREFIX` | `"/wm/"`, unchanged |
| `registerMicroservice(String id, String uri)` | unchanged |
| `registerMicroservices(Map<String, String> uris)` | unchanged, still registers only ids it does not know yet |

What changes is what the class is. It implemented `RouteLocator` and returned a `Flux<Route>`; it
implements `RouterFunction<ServerResponse>` and answers `route(ServerRequest)` per request. There
is no `RefreshRoutesEvent` in the WebMVC gateway, so nothing is published when a module registers.

The bean itself has to stay the same object for the lifetime of the application, because the
servlet gateway collects its `RouterFunction` beans once during startup and never asks the context
again. So the registry is one stable bean holding an `AtomicReference` to the router function it
currently delegates to. A registration rebuilds that function under the registry's own lock and
swaps the reference, and because `RouterFunctionMapping` calls `route(ServerRequest)` per request
without caching the outcome, the next request sees the new route.

A derived application is affected if it did one of these:

- published `RefreshRoutesEvent` itself, for instance after registering a module out of band. The
  event class no longer exists. Delete the publication; the registry needs no notification.
- implemented `RouteLocator` to add routes of its own. Those become `RouterFunction` beans.
  Spring picks up every `RouterFunction<ServerResponse>` bean in the context, and
  `RouterFunctionMapping` runs before `RequestMappingHandlerMapping`, so a router function wins
  against a controller mapped on the same path without any ordering configuration.
- configured gateway routes in YAML. Those keys move, see
  [configuration.md](configuration.md).

Two details of the WebMVC gateway are easy to get wrong when writing a route by hand. Only scheme,
host and port of the URI passed to `BeforeFilterFunctions.uri(...)` are used, so the path part of a
target URI has to end up in the replacement of `BeforeFilterFunctions.rewritePath(...)`. And a
module id can contain regular expression metacharacters, so quote it when building the pattern.

## Web infrastructure of your own

| Before | After |
|---|---|
| `WebFilter` | `jakarta.servlet.Filter`, usually `OncePerRequestFilter` |
| `WebFluxConfigurer` | `WebMvcConfigurer` |
| `CorsWebFilter` | `CorsFilter`, or the CORS configuration of the security DSL |
| `ServerWebExchange` | `HttpServletRequest` and `HttpServletResponse` |
| `WebClient` | `RestClient` |

`WebClient` is the one to check for outside the cockpit code as well, because business calls to
other services often use it and it disappears with `spring-webflux`. `RestClient` has the same
fluent shape and blocks instead of returning a publisher, so the call sites shrink rather than
grow.
