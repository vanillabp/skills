# Configuration

## `default-adapter` becomes `prioritized-adapters`

Version 1 named the adapter of a module. Version 2 names an order of adapters, because several
BPMS may serve one application, which is what makes BPMS migration a configuration change. The
key exists at the same three levels as before.

```yaml
# before
vanillabp:
  default-adapter: camunda7
  workflow-modules:
    ride:
      default-adapter: camunda7
      workflows:
        RideProcess:
          default-adapter: camunda7

# after
vanillabp:
  prioritized-adapters:
    - camunda7
  workflow-modules:
    ride:
      prioritized-adapters:
        - camunda7
      workflows:
        RideProcess:
          prioritized-adapters:
            - camunda7
```

The first entry starts new workflows and the others are asked for workflows started earlier. The
most specific non-empty value wins as a whole, so a workflow's list replaces the module's rather
than merging with it.

## Workflow modules are declared by a descriptor file

Version 1 knew three ways to name a workflow module: `spring.application.name` for
single-module applications, the name of the module's configuration file, or a static bean method
declaring the id together with the module's properties class.

```java
// version 1
@Bean
public static WorkflowModuleProperties ModuleProperties() {

    return new WorkflowModuleProperties(
            LoanApprovalProperties.class,
            LoanApprovalProperties.WORKFLOW_MODULE_ID);

}
```

Version 2 wants the id stated explicitly and in one place, so that a module travels with its own
artifact and is found the same way on both platforms. Add a text file whose content is the module
id:

```
src/main/resources/META-INF/workflow-module      # content: loan-approval
```

Delete the bean method afterwards. `WorkflowModuleProperties` no longer exists, so a project
keeping it does not compile. The properties class it named stays as it is, because binding a
module's own configuration has not changed.

One file per workflow module, placed in the artifact implementing that use case. An application
without any descriptor is treated as one single global module, so the file is what turns a use
case into a module of its own.

Keep the ids you had.

## Adapter settings move to `vanillabp.adapters.<id>.*`

In version 1 an adapter's settings lived per workflow module and its connection was configured
by the BPMS' own Spring integration. In version 2 an adapter id names one BPMS instance and
everything that instance needs lives in its own section, owned by VanillaBP, with no
BPMS-specific Spring configuration involved.

```yaml
# before (Camunda 8, connection via the Camunda Spring SDK)
camunda:
  client:
    mode: self-managed
    zeebe:
      grpc-address: http://localhost:26500
vanillabp:
  default-adapter: camunda8
  workflow-modules:
    ride:
      adapters:
        camunda8:
          task-timeout: PT5M

# after
vanillabp:
  prioritized-adapters:
    - camunda8
  adapters:
    camunda8:                 # the id; 'type' may be omitted because the id IS the type
      mode: self-managed
      rest-address: http://localhost:8080
      job-timeout: PT5M       # adapter-wide default, overridable per module, workflow and task
```

An id that is not an adapter type needs `vanillabp.adapters.<id>.type`, which is how two
instances of the same BPMS are told apart:

```yaml
vanillabp:
  prioritized-adapters:
    - saas          # new workflows start here
    - onprem        # existing workflows may still live here
  adapters:
    saas:
      type: camunda8
      mode: saas
      cluster-id: ...
    onprem:
      type: camunda8
      rest-address: http://camunda.internal:8080
```

The exact keys are the adapter's business. See the
[Camunda 7](https://github.com/camunda-community-hub/vanillabp-camunda7-adapter/wiki/Configuration),
[Camunda 8](https://github.com/camunda-community-hub/vanillabp-camunda8-adapter/wiki/Configuration)
and [Process-Engine-API](https://github.com/vanillabp/process-engine-api-adapter/wiki/Configuration)
wikis.

## Key by key

| Version 1 | Version 2 | Note |
|---|---|---|
| `vanillabp.default-adapter` | `vanillabp.prioritized-adapters` | a list now, at all three levels |
| `vanillabp.workflow-modules.<m>.default-adapter` | `vanillabp.workflow-modules.<m>.prioritized-adapters` | |
| `…workflows.<w>.default-adapter` | `…workflows.<w>.prioritized-adapters` | |
| `vanillabp.workflow-modules.<m>.adapters.<id>.resources-location` | unchanged but optional | derived as `classpath*:<module>/processes/<adapter>` |
| `camunda.bpm.*` (Camunda 7 engine) | `vanillabp.adapters.<id>.*` | the adapter runs the plain engine itself, so the Camunda Spring Boot starter is gone |
| `camunda.client.*`, `zeebe.client.*` (Camunda 8) | `vanillabp.adapters.<id>.*` | `mode`, `rest-address`, `grpc-address`, `prefer-rest-over-grpc`, `tenant-id`, `cluster-id`, `region`, `client-id`, `client-secret` |
| `…adapters.camunda8.task-timeout` | `job-timeout` | configurable at adapter, workflow-module, workflow and task level |
| `…adapters.camunda7.use-bpmn-async-definitions` | removed | the adapter forces `asyncBefore` and `asyncAfter` onto service-like tasks at parse time, so every task runs in its own transaction |
| `…adapters.<bpms>.tenant-id` | `vanillabp.adapters.<id>.tenant-id` | the tenant belongs to the BPMS instance now rather than to the workflow module, and without it the module id is used exactly as in version 1 |
| `…adapters.<bpms>.use-tenants: false` | `vanillabp.adapters.<id>.name-clash-avoidance: none` | one concept replaces it, see below |
| `vanillabp.allow-connectors`, `…retry-backoff` | not yet in version 2 | tell the VanillaBP team if you rely on them |
| `vanillabp.resilience.*` | removed | never consumed, and retry settings return per adapter with their first consumer |

## Keeping workflow modules apart: choose a mode

Version 1 isolated a workflow module by a BPMS tenant and let you switch that off per module.
Version 2 makes it one explicit setting with three values:

```yaml
vanillabp:
  adapters:
    camunda7:
      name-clash-avoidance: by-adapter   # tenant per workflow module, version 1's behavior
      # name-clash-avoidance: use-prefix # no tenant, VanillaBP prefixes the identifiers
      # name-clash-avoidance: none       # nothing is scoped, the old 'use-tenants: false'
```

Both Camunda adapters default to `by-adapter`, which is version 1's behavior, so an application
which ran version 1 with the defaults keeps its tenants without configuring anything. Where
`none` applies, the adapter logs a WARN per workflow module naming the alternatives, and
`accept-unscoped-identifiers: true` puts that decision on record.

Two settings that used to fit together now contradict each other and the boot says so: a
`tenant-id` without any level using `by-adapter` fails, because the deployment would ignore the
configured name. Where `by-adapter` does apply, Camunda 8 also asks the cluster whether
the tenant can be used at all before deploying.

Three notes for upgraders:

- Camunda 7: nothing to do where version 1 ran with its defaults. It deployed every workflow
  module into a tenant named after it, and `by-adapter` deploys into the same one, so the
  identifiers the engine answers by stay what they were.
- Camunda 7 and Camunda 8 with `use-tenants: false`: this is the case which needs a line. Those
  workflows carry no tenant, the default deploys into one, and nothing would be found any more.
  Set `name-clash-avoidance: none`.
- Camunda 8 without multi-tenancy: a cluster started from the stock image rejects a tenant id, so
  the boot ends with a message naming both ways out, `use-prefix` to keep the modules apart
  without a tenant and `none` where the identifiers are unique anyway. An application which ran
  an early VanillaBP 2 version, whose adapter put everything into the default tenant, belongs to
  the `none` case as well.
- Process-Engine-API: that BPMS has no tenants, so `by-adapter` cannot work there. The adapter
  fails the boot until you choose `use-prefix` or `none`. There was never a version 1 release of
  it, so this is not an upgrade question.

Switching to `use-prefix` changes the identifiers the BPMS sees, so it is a migration rather
than a property change. Read
[how name clashes are avoided](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-modules#how-name-clashes-are-avoided)
first.

## What you may now delete

An application with one adapter dependency and one workflow module needs no `vanillabp.*`
property at all, because the classpath is the configuration. The adapter section, the module
section and `resources-location` are all derived, and only what the BPMS itself needs, such as
a cluster address, remains. Everything you configured explicitly keeps working.

Where the application comes from a Camunda 8 adapter of 1.6.3 or earlier, two tables are left
behind as well: `CAMUNDA8_DEPLOYMENTS` and `CAMUNDA8_RESOURCES`, or two MongoDB collections of
the same names. That adapter kept its own record of what it had deployed so it could re-read the
models of older process versions; 1.7.0 dropped it and version 2 asks the cluster instead. After
the upgrade nothing reads them and they hold every BPMN the application ever deployed, so point
the user at them. Dropping them is the user's decision, not something to do for them.

## A new section for remote BPMS: the outbox

Adapters of remote BPMS (Camunda 8 and Process-Engine-API) start workflows in two phases
through a transaction outbox. It works out of the box, the store is created automatically, and
it is tunable via `vanillabp.outbox.*`. Read
[what it guarantees](https://github.com/vanillabp/adapter-platform-integration/wiki/Spring-Boot-integration#what-the-outbox-guarantees)
before going to production, because monitoring blocked entries is an operations duty.
