# Tests which prove each step

An application which starts after the upgrade has not shown that its workflows still behave.
Only a test which drives the workflows can show that. Find such tests before you change
anything, run them on version 1, and run them again after every step.

This is about VanillaBP. It is not a test plan for the whole application. Other parts of the
application may fail for reasons of their own. The tests here answer one question: does every
workflow still do what it did before the upgrade?

## What counts

A test counts if the upgrade leaves it as it is. Then a red result points at the upgrade and
not at the test. Such a test:

- drives the application from outside the VanillaBP classes. It goes through the application's
  own API, through its business services, or through `ProcessService` methods which keep their
  signature in version 2. The three removed overloads are no such methods, see
  [code-changes.md](code-changes.md#removed-processservice-overloads).
- checks outcomes the business cares about: the state of the aggregate in the database, the
  calls which reached a neighbour system (a stub or a simulator), a user task which can be
  reached and completed, a workflow which got to its end.
- runs against a real BPMS. That is the embedded Camunda 7 engine, or a Camunda 8 cluster from
  Testcontainers or from the project's existing test setup.
- names no adapter class, no type from `org.camunda.community.vanillabp` or from an adapter
  package of `io.vanillabp`, and no VanillaBP configuration class.

A test which reaches the workflows only indirectly counts too. Take a test of an endpoint which
starts a workflow on the side and waits for its result. It is exactly what you need.

A test suite of the whole application is the best choice, if there is one. Run it before and
after every step. Otherwise use the tests which reach the workflows.

## What does not count

A unit test which mocks VanillaBP classes, or calls a `@WorkflowTask` method directly, proves
nothing about the upgrade. The upgrade may rewrite it, because it names exactly what changes.
Keep these tests and fix them where they stop compiling. But do not rely on them.

## What moves although the test stays

The test code stays. Its resources move.

- `vanillabp.*` keys and keys of the BPMS in a test profile or a test resource move in step 3,
  like every other key. See [configuration.md](configuration.md). Test resources are where an
  upgrade is most often left half done.
- The BPMS setup of a test changes when it uses the BPMS' own Spring integration. Version 2 no
  longer uses it: the Camunda Spring Boot starter is gone, and on Camunda 8 no Spring
  configuration of the BPMS takes part. So keep that setup apart from the test methods, in a base
  class or a test configuration. Then the test methods stay as they are. On Camunda 8 the setup
  has to change, see the next section.
- A Camunda 8 test whose cluster is not up yet waits for it. The adapter gives the cluster
  `vanillabp.adapters.<id>.startup-wait` to answer, ten minutes by default. Start the container
  before the application context.

## A Camunda 8 test on Camunda Process Test

A version 1 application on Camunda 8 usually tests with Camunda Process Test:
`@SpringBootTest` together with `@CamundaSpringProcessTest`. Such a test does not run on version
2 as it is. Its test methods can stay. Its setup changes.

Camunda Process Test first loads the application context. Only after that does it
start its cluster in a container. It gives the address of that cluster to the client of Camunda's
Spring integration, not to VanillaBP. The Camunda 8 adapter of version 2 builds its own client
while the context starts, from `vanillabp.adapters.<id>.*`. At that moment Camunda Process Test
has no cluster yet. So there is no address you could hand over. The adapter connects to its default
address, `localhost:8088` on the 8.8 line and `localhost:8080` on 8.9 and 8.10. It waits for
`startup-wait`, and the test fails after ten minutes with "did not reach the cluster". If a
cluster of your own runs on that port, the test talks to that cluster instead, without a word.

So start the cluster before the context, and give its address to the adapter. The integration
tests of the Camunda 8 adapter do the same. Use the `camunda/camunda` image of your line,
and Testcontainers (`org.testcontainers:testcontainers` in test scope, Spring Boot manages the
version):

```java
@SpringBootTest
class OrderWorkflowTest {

  static final String H2 = "jdbc:h2:mem:camunda;DB_CLOSE_DELAY=-1;MODE=PostgreSQL";

  static final GenericContainer<?> CLUSTER = new GenericContainer<>("camunda/camunda:8.10.10")
      .withExposedPorts(8080, 26500, 9600)
      .withEnv("SPRING_PROFILES_ACTIVE", "broker,consolidated-auth,security")
      .withEnv("CAMUNDA_SECURITY_AUTHENTICATION_UNPROTECTEDAPI", "true")
      .withEnv("CAMUNDA_SECURITY_AUTHORIZATIONS_ENABLED", "false")
      // H2 as the secondary storage, so that the adapter can search the cluster
      .withEnv("CAMUNDA_DATABASE_TYPE", "rdbms")
      .withEnv("CAMUNDA_DATABASE_URL", H2)
      .withEnv("CAMUNDA_DATABASE_USERNAME", "sa")
      .withEnv("CAMUNDA_DATA_SECONDARYSTORAGE_TYPE", "rdbms")
      .withEnv("CAMUNDA_DATA_SECONDARYSTORAGE_RDBMS_URL", H2)
      .withEnv("CAMUNDA_DATA_SECONDARYSTORAGE_RDBMS_USERNAME", "sa")
      .withEnv("ZEEBE_BROKER_EXPORTERS_RDBMS_CLASSNAME", "io.camunda.exporter.rdbms.RdbmsExporter")
      .withEnv("ZEEBE_BROKER_EXPORTERS_RDBMS_ARGS_FLUSH_INTERVAL", "PT0S")
      .waitingFor(Wait.forHttp("/actuator/health/status").forPort(9600));

  static {
    CLUSTER.start();
  }

  @DynamicPropertySource
  static void camundaCluster(final DynamicPropertyRegistry registry) {
    registry.add("vanillabp.adapters.camunda8.rest-address",
        () -> "http://" + CLUSTER.getHost() + ":" + CLUSTER.getMappedPort(8080));
    registry.add("vanillabp.adapters.camunda8.grpc-address",
        () -> "http://" + CLUSTER.getHost() + ":" + CLUSTER.getMappedPort(26500));
  }

  // the test methods of version 1, unchanged
}
```

These are the settings Camunda Process Test gives its own container. Put the container and the
method into a base class. Then every test class gets the same cluster, and it starts once per
test run.

A test whose methods check what the business cares about needs nothing more. Then remove
`@CamundaSpringProcessTest` and the dependency on Camunda Process Test.

Some test methods use Camunda's assertions, `CamundaAssert` or `CamundaProcessTestContext`. For
those, keep `@CamundaSpringProcessTest`, and point it at the same container with its remote
runtime:

```java
registry.add("camunda.process-test.runtime-mode", () -> "remote");
registry.add("camunda.client.rest-address", () -> "http://" + CLUSTER.getHost() + ":" + CLUSTER.getMappedPort(8080));
registry.add("camunda.client.grpc-address", () -> "http://" + CLUSTER.getHost() + ":" + CLUSTER.getMappedPort(26500));
registry.add("camunda.process-test.remote.camunda-monitoring-api-address",
    () -> "http://" + CLUSTER.getHost() + ":" + CLUSTER.getMappedPort(9600));
```

Add `ZEEBE_CLOCK_CONTROLLED=true` to the container, so that Camunda Process Test can still set
the clock. One more thing then breaks. After each test method, Camunda Process Test deletes all
data of the cluster, and the deployments too. The adapter deploys a workflow module once, when
the context starts. So the first test method passes. The next one fails, because the cluster
answers "Expected to find process definition ... but none found". There are two ways around it:

- On 8.10, turn the deletion off with `camunda.process-test.data-deletion-mode: none`.
- On 8.8 and 8.9 there is no such key. Annotate the test class with
  `@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)`. Then each test
  method gets a new context, and the adapter deploys again. That costs a context start per test
  method.

This was tried with Spring Boot 4.1.2 against `camunda/camunda` 8.8.47, 8.9.29 and 8.10.10, with
the matching adapter line. The test support was `camunda-process-test-spring-boot-4` 8.8.47 and
`camunda-process-test-spring` 8.9.29 and 8.10.10, the artifacts Camunda's testing guide names for
[8.8](https://docs.camunda.io/docs/8.8/apis-tools/testing/getting-started/),
[8.9](https://docs.camunda.io/docs/8.9/apis-tools/testing/getting-started/) and
[8.10](https://docs.camunda.io/docs/apis-tools/testing/getting-started/). The 8.8 artifact is
documented for Spring Boot 4.0 and works on 4.1 as well.

A test on `@ZeebeSpringTest` or `@ZeebeProcessTest` has to move to Camunda Process Test first.
Camunda deprecated those in 8.8 and removed them in 8.10. That is Camunda's change, not
VanillaBP's. Follow Camunda's
[migration guide](https://docs.camunda.io/docs/apis-tools/migration-manuals/migrate-to-camunda-process-test/).
Do it in step 1, while the application is still on version 1.

On Camunda 7 none of this applies, because the adapter runs the engine inside the application.
A test with `camunda-bpm-spring-boot-starter-test` keeps its setup, except for the keys:
every `camunda.bpm.*` key in a test resource ends the start, like everywhere else. Camunda's
assertions find the engine on their own while there is only one. With two Camunda 7 adapter ids
there are two engines, and a test hands the right one over first, with
`BpmnAwareTests.init(processEngine)`. The adapter offers each engine as a bean of type
`ProcessEngine`, named `Camunda7_ProcessEngine_<adapter id>`.

## Finding the tests

Use whatever tool is there. VanillaBP 1 ran on Spring Boot only, so the tests are Spring Boot
tests.

| Look for | Command | Meaning |
|---|---|---|
| Tests with an application context | `grep -rln "@SpringBootTest" --include="*.java" --include="*.kt" .` | candidates. Read each one |
| A BPMS from a container | `grep -rln "testcontainers\|Testcontainers\|@Container" --include="*.java" --include=pom.xml .` | a Camunda 8 cluster for the tests, or a neighbour system |
| `ProcessService` in tests | `grep -rn "ProcessService" --include="*.java" src/test` | a test which starts a workflow, completes a task or correlates a message. Check that it uses none of the removed overloads |
| Workflow services in tests | `grep -rln "@WorkflowService" --include="*.java" .`, then grep `src/test` for each class name | a test which uses the class directly is most likely a unit test |
| Indirect callers | grep `src/test` for the endpoints, business services and messages which lead to `startWorkflow`, `completeTask`, `completeUserTask` and `correlateMessage` | tests which reach the workflows through the application |
| Mocks of VanillaBP | `grep -rn "ProcessService" --include="*.java" src/test \| grep -i "mock"` | a `ProcessService` which is a mock does not count |
| Adapter types in tests | `grep -rn "org.camunda.community.vanillabp\|io.vanillabp" --include="*.java" src/test \| grep -v "io.vanillabp.spi"` | a test which names an adapter or configuration class will change with the upgrade |

Then judge the coverage per BPMN process. For each process, which of these does a test drive?

- the start,
- every asynchronous task, at least one per process,
- every message the process waits for,
- every user task the application completes,
- the end.

Report the result to the user, one line per process. Name the tests, and name the gaps.

## When coverage is missing

Ask the user whether to write the missing tests. Say why: a start after the upgrade does not
prove that the workflows still behave. Without such tests, a broken API path or a task which
no longer runs shows up in production.

If the user says yes, write per process at least start, asynchronous task, message and end, as
far as the process has them. Follow what the upgrade touches. A process with a user task needs
one completed, and one which waits for a message needs that message correlated. Write them by
the rules of "What counts". They have to run green on version 1. Commit them before you change
anything else. Then the baseline is a commit of its own.

If the user says no, write that down in the report, and do the checks of step 5 by hand.

## Writing a test which holds for both versions

Version 2 changes some timings on purpose, see [behavior-changes.md](behavior-changes.md). A
test which asserts right after a call may pass on version 1 and fail on version 2, although
nothing broke.

- On a remote BPMS, `startWorkflow` writes an outbox entry, and the workflow is created after
  the commit, on another thread. On Camunda 7 every operation which moves a workflow on runs
  after the commit as well. And the Camunda 7 adapter makes every service-like task
  asynchronous, so it runs in its own transaction.
- So do not assert right after the call. Wait for the outcome, with a timeout. A library like
  Awaitility does that on both versions:

```java
String rideId = rideApi.requestRide(rideRequest);

await().atMost(Duration.ofSeconds(30)).untilAsserted(() ->
        assertThat(rides.findById(rideId)).get()
                .extracting(Ride::getStatus)
                .isEqualTo(RideStatus.CONFIRMED));
```

- Call `ProcessService` inside a transaction, the way the application does. Version 2 requires
  an active one for `startWorkflow`, `completeTask`, `cancelTask`, `completeUserTask`,
  `cancelUserTask` and `correlateMessage`. A test which goes through the application's API gets
  that transaction from the API.
- Let the BPMS run its jobs itself. A test which runs jobs by hand through the engine's API, or
  switches the job executor off with a `camunda.bpm.*` key, ties itself to the BPMS setup of
  version 1. Every `camunda.bpm.*` key ends the start in version 2.
- Find workflows and tasks by the aggregate or by the business key. Do not use process
  definition ids. In version 2 the viewer API returns them as `<adapter id>#<BPMS specific id>`.

Two behaviours of version 2 make a test red on purpose until somebody decides. The first start
refuses an aggregate which shares everything. That is decided in step 3. Removing
`@Transactional` from a class which is also the business service breaks the paths which start
at the API. That is step 4, and a test through the API is what shows it.

## Running them after each step

Run the baseline before step 1. It has to be green. A test which is red on version 1 already
gets named to the user, and stays out of the comparison.

After each step, run the same tests and tell the user the result:

| After | Run | Why |
|---|---|---|
| Step 1 | all tests | the application runs on VanillaBP 1 and Spring Boot 4.1 |
| Step 2 | compile main and test code only | the dependencies are version 2, the configuration is still version 1. The application does not start yet. A test which counts compiles without a change |
| Step 3 | all tests, once the application starts | a start which ends at a transaction annotation belongs to step 4. Then measure after step 4 |
| Step 4 | all tests | a combined class which lost its annotation breaks the API paths here |
| Step 5 | all tests | the tests are the first part of the verification. The checks by hand come after them |

A test which turns red stops the upgrade. Find out whether VanillaBP is the cause before you
go on. If it is, fix it in this step. If the cause lies outside VanillaBP, name the test and
the cause to the user. Do not fix it. Do not skip or hide it either. The user decides whether
the upgrade goes on.
