# The web layer

## Generated API interfaces

The OpenAPI specifications are unchanged. The generator changes. Its `reactive` option is off,
so the generated interfaces are plain Spring MVC.

Three things happen to every method:

1. The return type loses its wrapper. `Mono<ResponseEntity<X>>` becomes `ResponseEntity<X>`, and
   `Flux<X>` becomes `List<X>`.
2. A request body loses its wrapper too. `Mono<UserTasksRequest>` becomes `UserTasksRequest`.
3. The trailing `ServerWebExchange` parameter is gone.

So a controller written against the reactive interface turns from a chain into a body which
reads from top to bottom:

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

Do not copy signatures out of this file. First generate the API artifact again, or resolve it.
Let the compiler tell you what each interface method looks like. The order of the parameters,
and whether they may be null, follow the specification of the version you build against.

The construction you will unwind most often is `Mono.zip` of the user context and the request
body. It becomes two statements, and the `Tuple2` accessors `getT1()` and `getT2()` disappear
with it.

## Controllers extending the container

The container ships abstract controllers. A customized application extends them to plug in its
own queries. Among them are `AbstractUserTaskListGuiApiController`,
`AbstractWorkflowListGuiApiController` and `AbstractWorkflowModulesGuiApiController`. Their
abstract hook methods change the same way:

```java
// before
protected abstract Mono<Page<UserTask>> getUserTasks(
        UserDetails currentUser, int pageNumber, int pageSize, ...);

// after
protected abstract Page<UserTask> getUserTasks(
        UserDetails currentUser, int pageNumber, int pageSize, ...);
```

An override which keeps the old signature no longer overrides anything. If the hook is
abstract, the class fails to compile, and that is what you want. Where the hook was not
abstract, add `@Override` to every implementation of it before you start. Then the same mistake
is caught there too.

The injected fields of those base classes change type as well: `ReactiveUserContext` becomes
`UserContext`, described in [security.md](security.md).

## The update stream

`/gui/api/v1/updates` returns an `SseEmitter` instead of a `Flux<ServerSentEvent<?>>`.
`LoginApiController` holds one `UpdateEmitter` per subscribed browser in a map.

The event pipeline in front of it is unchanged. An application which publishes its own push
events keeps using `ApplicationEventPublisher` with `io.vanillabp.cockpit.gui.api.v1.GuiEvent`.
The container still collects those events through an
`@EventListener(classes = GuiEvent.class)`, and sends them to every emitter at a fixed
interval. So a derived application which only publishes `GuiEvent` has nothing to do here. One
which replaced the endpoint itself has to rewrite it. `LoginApiController` is the reference.

`SseEmitter` behaves differently from a `Flux`, in ways which only show up while the
application runs. Each open stream holds a request thread for as long as the browser keeps the
tab open. You can afford that because the threads are virtual. That is why
`spring.threads.virtual.enabled` is not optional. The emitter is created with a timeout of
`Long.MAX_VALUE`. Otherwise the default timeout of asynchronous requests would close a healthy
stream.

The container pings every subscriber every 27 seconds, so an idle stream stays open. It also
pings once 300 milliseconds after subscribing, so the client sees that the subscription stands.
Every timeout between the browser and the cockpit has to be longer than that interval, see
[behavior-changes.md](behavior-changes.md).

## Deep links into the single-page application

A path like `/tasklist/4711` matches no controller and no static resource. The application
shell has to be returned instead of an error. That used to be an `ErrorWebExceptionHandler`
which extended `AbstractErrorWebExceptionHandler`. Now `SpaNoHandlerFoundExceptionHandler` is a
`@ControllerAdvice` at `Ordered.HIGHEST_PRECEDENCE`. It has one `@ExceptionHandler` for
`NoResourceFoundException` and `NoHandlerFoundException`, which answers `200` with the shell
and `Cache-Control: max-age=0`.

The precedence makes it work. Otherwise the catch-all of `RestfulExceptionHandler` in `commons`
would turn every unknown path into an HTTP 500. A derived application with a
`@ControllerAdvice` of its own must not get in the way of either.

The reactive handler also rendered every other error status as JSON, because it sat in the
error path. The blocking one only answers the two not-found exceptions. Everything else goes
through the ordinary exception handling.

## JSON

`JsonConfiguration` in the container now only has its `JsonMapperBuilderCustomizer` beans. The
reactive codec beans it also declared are gone: `JacksonJsonEncoder`, `JacksonJsonDecoder`, and
the `WebFluxConfigurer` which registered them. Spring Boot builds the `JsonMapper` from the
customizers, and hands it to the message converter which Spring MVC uses. The format on the
wire is unchanged.

A derived application which registered codecs of its own drops them the same way, and keeps
only its customizers.

## The workflow module proxy

`MicroserviceProxyRegistry` keeps its public API:

| | |
|---|---|
| `WORKFLOW_MODULES_PATH_PREFIX` | `"/wm/"`, unchanged |
| `registerMicroservice(String id, String uri)` | unchanged |
| `registerMicroservices(Map<String, String> uris)` | unchanged, still registers only ids it does not know yet |

What the class is changes. It implemented `RouteLocator` and returned a `Flux<Route>`. Now it
implements `RouterFunction<ServerResponse>` and answers `route(ServerRequest)` per request. The
WebMVC gateway has no `RefreshRoutesEvent`, so nothing is published when a module registers.

The bean itself has to stay the same object for the whole life of the application. The servlet
gateway collects its `RouterFunction` beans once during startup and never asks the context
again. So the registry is one stable bean, which holds an `AtomicReference` to the router
function it currently hands over to. A registration rebuilds that function under the registry's
own lock and swaps the reference. `RouterFunctionMapping` calls `route(ServerRequest)` per
request and does not cache the result. So the next request sees the new route.

A derived application is affected if it did one of these:

- published `RefreshRoutesEvent` itself, for example after registering a module in some other
  way. The event class no longer exists. Delete the publication. The registry needs no
  notification.
- implemented `RouteLocator` to add routes of its own. Those become `RouterFunction` beans.
  Spring picks up every `RouterFunction<ServerResponse>` bean in the context.
  `RouterFunctionMapping` runs before `RequestMappingHandlerMapping`. So a router function wins
  against a controller mapped on the same path, without any configuration of the order.
- configured gateway routes in YAML. Those keys move, see
  [configuration.md](configuration.md).

Two details of the WebMVC gateway are easy to get wrong when you write a route by hand. Only
scheme, host and port of the URI passed to `BeforeFilterFunctions.uri(...)` are used. So the
path part of a target URI has to end up in the replacement of
`BeforeFilterFunctions.rewritePath(...)`. And a module id can contain special characters of
regular expressions. So quote it when you build the pattern.

## Web infrastructure of your own

| Before | After |
|---|---|
| `WebFilter` | `jakarta.servlet.Filter`, usually `OncePerRequestFilter` |
| `WebFluxConfigurer` | `WebMvcConfigurer` |
| `CorsWebFilter` | `CorsFilter`, or the CORS configuration of the security DSL |
| `ServerWebExchange` | `HttpServletRequest` and `HttpServletResponse` |
| `WebClient` | `RestClient` |

Check for `WebClient` outside the cockpit code as well. Business calls to other services often
use it, and it disappears with `spring-webflux`. `RestClient` has the same fluent shape, and
blocks instead of returning a publisher. So the places which call it get shorter, not longer.
