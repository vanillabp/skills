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

The reactive container shipped three abstract controllers, and also five concrete ones which
extended them. Both change, and not only in their types.

### The concrete list controllers stay in the container

From `0.8.0` on, the abstract classes are in the library `business-cockpit`. The five concrete
controllers stayed in `container`, and a derived application no longer has them on its classpath.
If the application relied on them, it starts, its user interface loads, and every list stays
empty. Nothing in the log says why. So the survey has to answer, for each of these five, whether
the application has a controller of its own on the same path:

| Controller in `container` | `@RequestMapping` | Base class | Returns |
|---|---|---|---|
| `tasklist.api.v1.GuiApiController` | `/gui/api/v1` | `AbstractUserTaskListGuiApiController` | `UserTaskVisibility.everythingTheUserMayWorkOn(currentUser)` |
| `tasklist.api.v1.CurrentUsersTasksGuiApiController` | `/gui/api/v1/current-user` | `AbstractUserTaskListGuiApiController` | `UserTaskVisibility.onlyWhatIsTheUsersOwn(currentUser)` |
| `tasklist.api.v1.UsersRoleAssignedTasksGuiApiController` | `/gui/api/v1/user-roles` | `AbstractUserTaskListGuiApiController` | `UserTaskVisibility.whatTheUsersGroupsMayTake(currentUser)` |
| `workflowlist.api.v1.GuiApiController` | `/gui/api/v1` | `AbstractWorkflowListGuiApiController` | `WorkflowVisibility.workflowsAddressedTo(currentUser)` |
| `workflowmodules.api.v1.GuiApiController` | `/gui/api/v1` | `AbstractWorkflowModulesGuiApiController` | `WorkflowModuleVisibility.modulesAddressedTo(currentUser)` |

The packages start with `io.vanillabp.cockpit`. For each one the application lacks, write a
subclass of a few lines. Copy it from `container`, and give it a bean name of its own, because
three of the classes have the same simple name. Ask the user which lists their user interface
calls, and do not write a controller nobody calls.

The workflow list of the reactive container showed every workflow to everybody. The one in
`container` now shows a user the workflows addressed to them or to one of their groups, and the
ones addressed to nobody. If the users of the application have to see every workflow, return
`WorkflowVisibility.everyWorkflow()`. Ask the user before you choose.

### A subclass answers with a visibility

The reactive abstract classes had one abstract hook per read and per action, for example
`getUserTasks`, `getUserTask`, `claimTask`, `getWorkflows` and `getWorkflowModules`, and every
subclass implemented all of them. Now each class has one abstract method, and it returns a
visibility:

| Base class | Abstract method | Returns |
|---|---|---|
| `AbstractUserTaskListGuiApiController` | `userTasksVisibleTo(UserDetails currentUser)` | `UserTaskVisibility` |
| `AbstractWorkflowListGuiApiController` | `workflowsVisibleTo(UserDetails currentUser)` | `WorkflowVisibility` |
| `AbstractWorkflowModulesGuiApiController` | `workflowModulesVisibleTo(UserDetails currentUser)` | `WorkflowModuleVisibility` |

The base class answers the list, the search, the count, a single entry and every action on one
from that visibility. A converted subclass looks like this:

```java
// before
@Override
protected Mono<Page<UserTask>> getUserTasks(
        UserDetails currentUser, int pageNumber, int pageSize, ...) {
    return userTaskService.getUserTasks(...);
}
// and the same for getUserTasksUpdated, kwic, getUserTask, markAsRead,
// claimTask, claimTasks, assignTask, assignTasks and setFollowUpDate

// after
@Override
protected UserTaskVisibility userTasksVisibleTo(
        final UserDetails currentUser) {

    return UserTaskVisibility.everythingTheUserMayWorkOn(currentUser);

}
```

Do not translate the old hooks one by one into blocking code. First find out which users and
groups each hook let through. Then pick the visibility which says the same, or build one of your
own from the record's constructor. Show the user the visibility you chose for each controller,
because a different rule shows other people other tasks.

What happens to the old hooks:

- `getUserTask`, `markAsRead`, `claimTask`, `claimTasks`, `assignTask`, `assignTasks`,
  `setFollowUpDate`, `getWorkflow` and `getWorkflowModules(UserDetails)` are no longer hooks.
  Delete them. A method left behind without `@Override` still compiles and is never called.
- `getUserTasks`, `getUserTasksUpdated` and `kwic` of the task list, and `getWorkflows`,
  `getWorkflowsUpdated`, `getUserTasksOfWorkflow` and `kwic` of the workflow list, are no longer
  abstract. Delete them too, unless the application reads the data in a way of its own. An override
  takes the visibility as its first parameter instead of the current user, and returns `Page` or
  `List` instead of `Mono` or `Flux`. `countUserTasks` and `countWorkflows` are new hooks of the
  same kind.

Add `@Override` to every method you keep, before you start. Then a hook whose parameters changed
fails to compile, instead of silently overriding nothing. Take the parameters from the class on the
classpath, not from this file. They changed after `0.8.0` too, and the release notes of the
`business-cockpit` repository list each change.

The injected fields of those base classes change type as well: `ReactiveUserContext` becomes
`UserContext`, described in [security.md](security.md).

## The update stream

`/gui/api/v1/updates` returns an `SseEmitter` instead of a `Flux<ServerSentEvent<?>>`.
`LoginApiController` holds one `UpdateEmitter` per subscribed browser in a map.

An application which publishes its own push events keeps using `ApplicationEventPublisher` with
`io.vanillabp.cockpit.gui.api.v1.GuiEvent`. Its constructor changed. It was
`GuiEvent(source, targetGroups, event)`, and it is `GuiEvent(kindOfEntity, entityId, event)` now.
The first value is the name of the event in the stream, for example `UserTask`. The cockpit no
longer sends every event to every stream. For the kinds `UserTask` and `Workflow` it decides by
the id, with the visibility of the list controllers above. An event of another kind, and an event
without an id, still reaches every stream. An application which replaced the endpoint itself has to
rewrite it. `LoginApiController` in the library is the reference.

What the browser reads changed too. An element of the stream no longer has `targetGroups`, and it
names `workflowModuleId`. `type` has the new value `RELOAD`, which tells the browser to load every
list of that kind again. A stream ends when the sign-in expires. If the application brings its own
user interface, the release notes of the `business-cockpit` repository say what it has to follow.

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
