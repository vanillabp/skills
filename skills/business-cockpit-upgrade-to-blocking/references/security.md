# Security

## Types

| Before | After |
|---|---|
| `SecurityWebFilterChain` | `SecurityFilterChain` |
| `ServerHttpSecurity` | `HttpSecurity` |
| `@EnableWebFluxSecurity` | `@EnableWebSecurity` |
| `@EnableReactiveMethodSecurity` | `@EnableMethodSecurity`, which the container declares as `@EnableMethodSecurity(securedEnabled = true)` |
| `ServerWebExchangeMatcher` | `RequestMatcher` |
| `PathPatternParserServerWebExchangeMatcher` | `PathPatternRequestMatcher`, built through its static `pathPattern(...)` rather than with `new` |
| `OrServerWebExchangeMatcher` | `OrRequestMatcher` |
| `authorizeExchange(...)`, `anyExchange()` | `authorizeHttpRequests(...)`, `anyRequest()` |
| `MapReactiveUserDetailsService` | `InMemoryUserDetailsManager` |
| `UserDetailsRepositoryReactiveAuthenticationManager` | `new ProviderManager(new DaoAuthenticationProvider(userDetailsService))` |
| `HttpBasicServerAuthenticationEntryPoint` | `BasicAuthenticationEntryPoint` |
| `NoOpServerSecurityContextRepository.getInstance()` | `new NullSecurityContextRepository()` |
| `SecurityWebFiltersOrder.HTTP_BASIC` | `BasicAuthenticationFilter.class` as the anchor of `addFilterAfter` |
| `WebFilter` | `jakarta.servlet.Filter`, usually `OncePerRequestFilter` |

`PathPatternRequestMatcher` lives in
`org.springframework.security.web.servlet.util.matcher`, and the container static-imports its
`pathPattern` factory method.

`securityMatcher(...)` keeps its name and still takes a single `RequestMatcher`, so the BPMS API
chain scopes itself the same way it did before, with an `OrRequestMatcher` around two path
patterns.

`BasicAuthenticationEntryPoint` is instantiated with `new` and configured with `setRealmName(...)`,
where the reactive entry point had `setRealm(...)`. Nothing calls its `afterPropertiesSet()` when
it is built by hand rather than declared as a bean, so the container calls it itself before handing
the instance to the DSL.

The lambda DSL is unchanged in shape, so a chain converts statement by statement:

```java
// before
@Bean
@Order(99)
@ConditionalOnMissingBean(name = "guiHttpSecurity")
public SecurityWebFilterChain guiHttpSecurity(
        final ServerHttpSecurity http, ...) {

    http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchanges -> exchanges
                    .matchers(appInfoMatcher, assetsMatcher).permitAll()
                    .anyExchange().authenticated())
            .addFilterAfter(jwtSecurityFilter(...), SecurityWebFiltersOrder.HTTP_BASIC);
    return http.build();

}

// after
@Bean
@Order(99)
@ConditionalOnMissingBean(name = "guiHttpSecurity")
public SecurityFilterChain guiHttpSecurity(
        final HttpSecurity http, ...) throws Exception {

    http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(requests -> requests
                    .requestMatchers(appInfoMatcher, assetsMatcher).permitAll()
                    .anyRequest().authenticated())
            .addFilterAfter(jwtSecurityFilter(...), BasicAuthenticationFilter.class);
    return http.build();

}
```

`HttpSecurity.build()` throws a checked `Exception`, which the reactive builder did not, so every
converted bean method gains a `throws Exception`.

Where the reactive chain configured the authentication manager inside `httpBasic(...)`, the servlet
chain sets it on the chain itself with `authenticationManager(...)`. That keeps it scoped to the
one chain, which is what the container relies on so that the GUI chain and the BPMS API chain
authenticate against different credentials.

`container/README.md` in the `business-cockpit` repository shows a full derived `guiHttpSecurity`
bean written against the blocking API. Read it before writing one of your own.

## The public matchers are renamed

`WebSecurityConfiguration` exposes the paths its chain leaves unprotected as public constants, and
a derived chain reuses them. Their names carry the old type and change with it:

| Before | After |
|---|---|
| `appInfoWebExchangeMatcher` | `appInfoRequestMatcher` |
| `currentUserWebExchangeMatcher` | `currentUserRequestMatcher` |
| `assetsWebExchangeMatcher` | `assetsRequestMatcher` |
| `staticWebExchangeMatcher` | `staticRequestMatcher` |
| `workflowModulesProxyWebExchangeMatcher` | `workflowModulesProxyRequestMatcher` |

The paths behind them are unchanged. This is the one rename in the upgrade that breaks a derived
application on a name rather than on a type, which is why the survey greps for the old names.

## JWT

The reactive JWT classes leave `commons`.

| Removed | Replacement |
|---|---|
| `JwtSecurityWebFilter` | `PassiveJwtSecurityFilter`, which existed in the reactive line already |
| `JwtServerSecurityContextRepository` | `JwtSecurityContextRepository`, the servlet equivalent |
| `ReactiveJwtUserDetailsProvider` | `JwtUserDetailsProvider` |
| `ReactiveUserDetailsProvider` | `io.vanillabp.cockpit.commons.security.usercontext.UserDetailsProvider` |
| `JwtBearerTokenConverter` | nothing. It was a `ServerAuthenticationConverter` for bearer tokens which the container never wired up, and it is deleted without a successor |

`JwtMapper`, `JwtAuthenticationToken`, `JwtAuthenticationTokenMapper`, `JwtProperties`,
`JwtUserDetails`, `JwtCookie` and `JwtLogoutSuccessHandler` keep their names, so a customization
which only supplies its own `JwtMapper` keeps working.

The work is split over two classes, one for each direction of the cookie.

`JwtSecurityContextRepository` writes it. It is a `SecurityContextRepository` wired into
`httpBasic(basic -> basic.securityContextRepository(...))`, and its `saveContext` adds the JWT
cookie to the response of the request which authenticated by basic auth. Its `loadContext` reads
the cookie back, which the interface still requires even though Spring Security marks the method
deprecated. Its static `clearCookie(JwtProperties, HttpServletResponse)` overwrites the cookie with
an expired one, and `JwtLogoutSuccessHandler` calls it.

`PassiveJwtSecurityFilter` reads it. It is an `OncePerRequestFilter` taking `JwtProperties` and a
`JwtMapper`, added after `BasicAuthenticationFilter`. It puts the authentication into
`SecurityContextHolder` and clears the context again in a `finally` block. Add it to the chain with
`addFilterAfter(...)` and do not declare it as a bean: Spring Boot would then register it a second
time as a plain servlet filter, outside the security filter chain.

`JwtLogoutSuccessHandler` is a `SimpleUrlLogoutSuccessHandler` now. It clears the cookie and
delegates to the base class, so the target URL is configured with `setDefaultTargetUrl(...)` rather
than passed to the constructor.

## Beans a customized application overrides

The container declares these conditionally, so a bean of the derived application takes their
place. Two of them back off on the bean name, so keep the name spelled exactly as it is.

| Bean name | Backs off on | Type before | Type after |
|---|---|---|---|
| `guiHttpSecurity` | the name | `SecurityWebFilterChain` | `SecurityFilterChain` |
| `userDetailsProvider` | the name | `ReactiveUserDetailsProvider` | `UserDetailsProvider` from `commons.security.usercontext` |
| `jwtMapper` | the type | `JwtMapper<? extends JwtAuthenticationToken>` | unchanged |

The container also declares `jwtSecurityContextRepository` unconditionally, and a derived
`guiHttpSecurity` bean can take it as a parameter instead of building one.

The chain protecting the BPMS API, `bpmsApiHttpSecurity` in `BpmsApiWebSecurityConfiguration`, is
declared unconditionally and is imported by the container's `WebSecurityConfiguration`. Its return
type changes with the rest, but it is not an extension point: a derived application which needs a
different rule for `/bpms/api/**` adds a chain of its own with a higher precedence rather than
replacing this one.

Watch out for the two interfaces called `UserDetailsProvider`. The one in
`io.vanillabp.cockpit.commons.security.usercontext` turns an `Authentication` into the details of
the caller, and that is the one the `userDetailsProvider` bean supplies. The one in
`io.vanillabp.cockpit.users` is the user directory, with `findUsers`, `getAllUsers` and `getUser`,
and it is unaffected by this upgrade. A derived application usually implements the second and
overrides the first, or the other way round, and the import line is the only thing distinguishing
them.

Chain ordering is unchanged in principle: the BPMS API chain is matched first and the GUI chain
runs at `@Order(99)`, so a derived chain slots in with an order between them or after them, as
before.

## Recording a login

The container upserts the `users` document of the caller when the single-page application asks for
the current user. That hook used to be `UserLoginUpsertWebFilter`, a `WebFilter`; it is
`UserLoginUpsertFilter`, an `OncePerRequestFilter`, now. Both are registered as a plain filter bean
rather than inside a security chain, deliberately, so that a derived application declaring its own
`guiHttpSecurity` still records logins. The bean is `@ConditionalOnMissingBean`, so an application
which replaced the reactive filter has to replace the servlet one instead, and the bean method in
`UserLoginUpsertConfiguration` is named `userLoginUpsertFilter` now.

`UserLoginUpsertService.upsertOnLogin` returns `void` instead of `Mono<Void>` and swallows its own
failures, as before.

## User context

`ReactiveUserContext` and `ReactiveUserContextConfiguration` are removed, together with the whole
`io.vanillabp.cockpit.commons.security.usercontext.reactive` package. `UserContext` stays, which is
the class `ReactiveUserContext` extended, so its methods are the ones that were there all along.

| Before | After |
|---|---|
| `ReactiveUserContext` | `UserContext` |
| `getUserLoggedInAsMono()` | `getUserLoggedIn()` |
| `getUserLoggedInDetailsAsMono()` | `getUserLoggedInDetails()` |
| bean `reactiveUserContext` | bean `userContext` |

Both blocking methods throw `BcUnauthorizedException` when there is no authentication and return
`null` for an anonymous caller, so a `switchIfEmpty` in the reactive version becomes a null check.
The reactive methods declared the same exception but rarely threw it, because an absent security
context made the chain empty before the check was reached. See
[behavior-changes.md](behavior-changes.md) for what that changes on the wire.

`UserContextConfiguration` declares the `userContext` bean and is conditional on a servlet web
application, which the application now is. An injection point typed `ReactiveUserContext` fails to
resolve after the upgrade, which is the loudest way this shows up.
