# How a VanillaBP application is laid out

This repeats the structure every blueprint follows. The version which decides lives at
`https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`, and it wins
where the two disagree.

## Placeholders

Every blueprint uses the same placeholders. Only the Maven coordinates differ, because they
carry the name of the blueprint. Replace all of them consistently. Most startup errors in
generated applications come from replacing one and forgetting another.

| Placeholder | Meaning | Example |
|---|---|---|
| `blueprint.workflowmodule` | base package | `com.acme.orders` |
| `blueprint.application` | package of the application, not above the workflow module | `com.acme.orders.application` |
| `loanapproval` | use case identifier as a Java package | `shipment` |
| `loan-approval` | use case identifier in kebab case: workflow module ID, resource directory, REST path, Maven module, `finalName` | `shipment` |
| `loan_approval` | BPMN process ID | `shipment` |
| `LoanApproval` | use case identifier in CamelCase: the start of class names, for example `LoanApprovalProperties`, `LoanApprovalIT` or the generated `LoanApprovalApi` | `Shipment` |
| `LOAN_APPROVAL` | use case identifier in upper snake case: table and collection names | `SHIPMENT` |
| `io.vanillabp.blueprint.<blueprint-id>-<platform>` | Maven groupId of every POM | `com.acme.orders` |
| `<blueprint-id>` | Maven artifactId of the root POM | `orders` |

The AGENTS.md of each blueprint lists the same placeholders with its own Maven coordinates.
Words of the domain, such as `loanRequestId` or `loanApprovals`, are no placeholders. Rename
them as your domain needs.

The BPMN process ID in the model has to match the one used in the code. The resource directory
has to match the workflow module ID.

## Reference structure

```
<base-package>.<usecase>
├── ApiController.java               <- driving adapter: HTTP calls in, implements the
│                                       interface generated from the OpenAPI file
├── Service.java                     <- business code, never touches VanillaBP
├── Workflow.java                    <- outgoing: the application tells the process
├── WorkflowTaskHandler.java         <- incoming: the process tells the application
├── config/<UseCase>Properties.java
└── model/
    ├── Aggregate.java
    └── AggregateRepository.java
```

An application built from several blueprints stays readable only if all of them agree on this
layout. So keep it.

That is the layout of one use case of medium size. A large process is split by section, and a
user task gets a package of its own. Both are described under
[rule 8](rules.md#8-one-primary-bpmn-process-per-workflow-aggregate). The classes keep their
names and their roles in every one of those packages.

## The Maven layout comes from the `module-*` blueprints

The package tree above is the inside of one workflow module. The Maven modules around it are
not always the ones of `module-single`. The blueprints `module-multi`, `module-interaction`,
`module-simulator` and `module-process-driver` change the Maven layout: ports and adapters in
modules of their own, an `<name>-api` module per API, a simulator with its own `api` and
`application`, a `test-driver` module, one module per workflow module.

The field `base` of the catalogue says which blueprint an entry builds on. `module-process-driver`
builds on `module-simulator`, which builds on `module-single`. Take the layout of the taken
blueprint which is deepest in that chain. Never start from `module-single` and add the other
modules from memory.

This is the layout of `module-process-driver`, the deepest of them today. Read it from the
delivered repository all the same, because it may have changed:

```
<root>
├── application/                       the Spring Boot or Quarkus application. It pulls in the
│                                        workflow module, the adapters of the ports, the BPMS
│                                        adapter and, in the Maven profile test-driver only,
│                                        the test driver
├── <use-case>/
│   ├── workflow-module/               the workflow module. It knows only the ports
│   └── test-driver/                   plays the users through the GUI API; production never
│                                        pulls it
├── external-systems/
│   └── <purpose>/                     one folder per port, named after what it is used for,
│       │                                for example e-signature
│       ├── port/                      the Java interface the workflow module calls
│       ├── <purpose>-api/             the OpenAPI file of the system
│       │   ├── client/                generated, used by the adapter
│       │   └── server/                generated, implemented by the simulator
│       └── <system>-adapter/          implements the port with the generated client, named
│                                        after the system it talks to, for example sign-now
└── simulator/
    ├── api/                           the test API of the simulator, with client and server
    └── application/                   plays every surrounding system
```

The blueprints name the adapter `rest-adapter`, because their systems are invented. In your
project the adapter carries the name of the real system.

Write this tree into the plan, as
[Planning a new application](working-in-stories.md#planning-a-new-application) says. Story 1
builds all of it, also the modules which stay nearly empty at first.

## Every API is an OpenAPI file, and all code is generated from it

An API starts as an OpenAPI file. Its paths, parameters and answers are written there. The build
generates the Java interfaces from it with the `openapi-generator-maven-plugin`, and the
TypeScript client of a user interface with the generator its blueprint names. Nobody writes a
REST client by hand. That means no `RestTemplate`, `RestClient`, `WebClient` or `fetch` call with
a path in it, no `@HttpExchange` or `@Path` interface, and no record for a request or an answer.
This holds in the backend, in the frontend, in the test driver, in the simulator and in the
tests. A server implements the generated interface. A caller injects the generated client.

`ApiController` implements the generated interface and holds the code of each call. It carries no
path and no parameter annotation of its own.

Where the file lives depends on who calls the API:

- Only people in a browser, or only the module's own user interface:
  `src/main/resources/<workflow-module-id>/openapi/<use-case>.yaml` in the module of the
  controller, like every other resource of the module. Its tag is the use case, so the interface
  is `LoanApprovalApi` in the package `<use case package>.api`. The user interface generates its
  TypeScript client from the same file.
- Any Java code, such as a system around the application, the simulator, the test driver or
  another module: a Maven module `<name>-api` with the file and two submodules, `client` and
  `server`. Each of them generates its classes from the file.

### Client and server are seen from the caller

`client` is what the caller uses. `server` is what the other side implements. Which side is
which follows from who calls whom, not from what is the application:

| API | `server` is used by | `client` is used by |
|---|---|---|
| GUI API of a workflow module | the workflow module, its `ApiController` | the test driver |
| API of a surrounding system | the simulator | the adapter which implements the port |
| Test API of the simulator | the simulator | the test driver and the tests |

A `client` module is ready to use. The application pulls in the module, sets the URL and the
other properties in `application.yaml` and injects the generated interface. Nothing else. On
Spring Boot the `client` module brings an auto-configuration for that. On Quarkus the generated
interface is a REST client of Quarkus, which finds it while it builds. The `client` modules of
`module-simulator` show both.

### A callback turns the two around

Some systems call back, for example an e-signature service when the contract is signed. The
callback is a second OpenAPI file in the same `<name>-api` module, and it goes the other way:

- `client` also carries the generated server interface of the callback. The adapter implements it
  with its callback controller.
- `server` also carries the generated client of the callback. The simulator uses it to call back.

So the side which calls the system also receives its callbacks, and the simulator does both for
the system. The `e-signature` module of `module-simulator` shows it.

### A surrounding system is a port with adapters

This is hexagonal architecture. A port is the inside, an adapter the outside.

- The *port* is a Java interface the workflow module owns. The business code calls it. It lives in
  a Maven module `port` of its own and is named after what the process uses it for, not after the
  system: `e-signature`, `credit-rating`. The port has no OpenAPI file.
- The *adapter* implements the port with the generated client. It is named after the system it
  talks to: `sign-now-adapter` for the e-signature port. A second system for the same purpose is a
  second adapter of the same port.
- The workflow module depends on the port only, never on an adapter or a `client` module. The
  application depends on the adapter. Tests of the workflow module may pull in the adapter and the
  simulator in the test scope.

The blueprint `bpmn-service-task` shows the first step: the port `CreditRatingClient` with a
`Local...` stand-in and a simulator in the tests. `module-simulator` shows the whole way: port,
adapter, `<name>-api` module and simulator. If nobody can tell you the API of the system yet, for
example because the BPMN only names it, keep the port with its stand-in and say so. Do not
invent an OpenAPI file for a system nobody described.

The generators and their settings for both platforms are in
[rule 7 of the blueprints' CONTRIBUTING.md](https://github.com/vanillabp-blueprints/blueprints/blob/main/CONTRIBUTING.md#rule-7-every-api-is-written-as-an-openapi-file).
The blueprints `module-single` and `module-simulator` show both kinds of file.

## One class per direction

Talking to a BPMS (business process management system) goes in both directions, and the two are
different things in the architecture:

```
ApiController ─────────────────┐
                               ├──→ Service ──→ Workflow ──→ ProcessService     outgoing
BPMS ──→ WorkflowTaskHandler ──┘                                         incoming
```

`Workflow` is the outgoing half. `ProcessService` is injected here and nowhere else. `Service`
calls in and names what happened in business terms. This class translates that into what the
process needs.

`WorkflowTaskHandler` is the incoming half. It carries `@WorkflowService` and every
`@WorkflowTask` method, and it calls `Service`. It is a driving adapter, the same kind of thing
as `ApiController`. That the caller is a BPMS and not a browser changes nothing.

```java
// Service.java - what happened
public void submitRiskAssessment(final String id, final boolean acceptable) {
  final var loanApproval = loanApprovals.findById(id).orElseThrow();
  loanApproval.setRiskAcceptable(acceptable);
  workflow.riskAssessmentSubmitted(loanApproval);
}

// Workflow.java - what it means for the process
public void riskAssessmentSubmitted(final Aggregate loanApproval) {
  bpms.correlateMessage(loanApproval, "RiskAssessed");
}

// WorkflowTaskHandler.java - what the process wants from the application
@WorkflowTask
public void assessRisk(final Aggregate loanRequest, @TaskId final String taskId) {
  loanApproval.riskAssessmentRequested(loanRequest, taskId);
}
```

Name the methods of `Workflow` after the business event (`riskAssessmentSubmitted`), never
after the BPMN element (`correlateRiskAssessedMessage`). The model may be remodelled. A message
may become a timer, and a task a call activity, and the business code does not notice. That is
the whole point.

Name an injected bean after the role it plays in the class, not after its type. The `Service`
of a loan approval is `loanApproval`, `ProcessService` is `bpms`, `Workflow` is `workflow`, and
a repository is named after what it stores, `loanApprovals`. A call then reads like a sentence
about the business case. Where a parameter would hide the field, give it a name from the
domain, as `loanRequest` above.

Do not merge the two classes. If you put both directions into one class, it depends on
`Service` while `Service` depends on it. Spring Boot rejects such a circular bean reference at
startup, unless you work around it with `@Lazy`. An interface which `Service` implements does
not help either, because the cycle is between beans, not between types. Splitting by direction
removes the cycle instead of hiding it.

Keep both classes even where the translation is a single line. The seam costs nothing while a
process is simple. It keeps the business code readable once the process is not simple any more.

If you want to unit-test the business service without a BPMS, let `Workflow` implement an
interface which the application owns, and inject that. Do it on the outgoing side only. The
incoming adapter may depend on the application directly, exactly as `ApiController` does.

## A `@WorkflowTask` method contains no business logic

It translates and does nothing else. The BPMS delivers the aggregate, `@TaskId`, `@TaskEvent`
and the multi-instance element with its index. The method turns that into a call to `Service`.
With a single service task that leaves one line. That is honest, because there is nothing to
translate. In a multi-instance task or a user task it becomes real work: pick the element this
call is about, keep the task ID, react when the task was canceled. That work belongs there, not
in the business code.

The work behind a task is business code and lives in `Service`. The handler may call it,
because the directions are split. Logic about the business object itself may sit on the
aggregate, which is a normal entity. But that is a matter of taste, not a rule. The rule is
that nothing computes inside a `@WorkflowTask` method.

## Two namespaces per workflow module

Workflow modules are not isolated by classloaders. They end up in one runtime, on one
classpath. So two modules have to differ in two namespaces. Both are derived from the same
identifier.

| | Rule | Example |
|---|---|---|
| Classes | a unique Java package for the whole module | `com.acme.orders.shipment` |
| Resources | every resource in one subdirectory named after the workflow module ID | `src/main/resources/shipment/…` |

Every resource means every one: BPMN models in `<module-id>/processes/<adapter-id>/`, the
module's configuration file `<module-id>/<module-id>.yaml`, templates, documents, schemas. A
resource at the classpath root works fine until a second workflow module ships a file of the
same name. Then one of them wins, without a word.

The single exception is the marker file `META-INF/workflow-module`, which contains the module
ID. It has to sit at that exact path, because that is how the module is recognised at all.

## Configuration per BPMS goes into a profile file

`application.yaml` of the application module holds what every engine needs. Everything which
belongs to one engine goes into the profile file of that engine, `application-camunda7.yaml`
and `application-camunda8.yaml`. Where a test needs it, it also goes into the workflow module's
test resources. Never mix the two. A reader has to be able to see what changes when the engine
changes. And an application which will soon migrate from one BPMS to another runs with both
profiles side by side.

The BPMS is named once, on the Maven command line. Every BPMS profile of a blueprint sets the
property `bpms`. The build filters it into `application.yaml` as `spring.profiles.active` or
`quarkus.config.profile.parent`. Surefire and failsafe pass it to the tests. Keep that wiring
when you copy a blueprint. Otherwise the engine has to be named twice, and the two will drift
apart.

If you name an adapter id whose adapter is not on the classpath, VanillaBP refuses to start,
because that is a configuration error. So a profile file never applies to the wrong engine.

## What an application needs to declare

A workflow module depends on `io.vanillabp:vanillabp-spring-boot-support` or
`io.vanillabp:vanillabp-quarkus-support`. They bring the business SPI (service provider
interface) along, and they expose no BPMS API, on purpose. The adapter dependency belongs in
the application module. On Quarkus both VanillaBP and the adapter are Quarkus extensions, so
both are explicit dependencies there.

Adapter configuration lives at `vanillabp.adapters.<id>.*`, where the id names one BPMS
instance. An application with one adapter dependency and one workflow module needs no
`vanillabp.*` property at all. The classpath is the configuration, and everything else is
derived from it.

## What is different on Quarkus

The structure, the class names and the VanillaBP annotations are the same on both platforms.
Each blueprint exists as a Spring Boot twin and a Quarkus twin, and the twins differ only where
the platform forces it. If you build on Quarkus, take the Quarkus twin of every blueprint and do
not port code from a Spring Boot twin. These are the places where the platform shows:

| Topic | Quarkus |
|---|---|
| Parent POM | imports `io.quarkus:quarkus-bom` and `io.vanillabp:vanillabp-bom` next to each other |
| Workflow module | `io.vanillabp:vanillabp-quarkus-support` and the `jandex-maven-plugin`. Without the index, nothing finds the module's `@WorkflowService` class |
| Application module | `io.vanillabp:vanillabp-quarkus-integration`, one adapter `io.vanillabp:<bpms>-adapter-quarkus` with its own version, `quarkus-config-yaml` for the YAML files, and the `quarkus-maven-plugin` with the goals `build`, `generate-code` and `generate-code-tests` |
| Booting | no `Application` class, no auto-configuration and no `TestApplication`. Quarkus finds the beans of every indexed JAR on its own |
| Beans | `@ApplicationScoped` and `@Inject`, not `@Service` or `@Component` |
| Repository of the aggregate | a Panache repository, for example `PanacheRepositoryBase<Aggregate, String>` with `@ApplicationScoped`. Panache Mongo, an active record and Spring Data are recognised as well. For anything else write a bean which implements `AggregatePersistenceAware` |
| Configuration of a module | an interface with `@ConfigMapping(prefix = "<module-id>")` and `@WithDefault`, not a class. The file `<module-id>/<module-id>.yaml` is loaded as defaults, and `application.yaml` wins over it |
| Transactions | `jakarta.transaction.Transactional` with `dontRollbackOn = TaskException.class`, where Spring says `noRollbackFor` |
| BPMS profile | the build filters it into `quarkus.config.profile.parent: "@bpms@"`. Filter only `application*.yaml`, with `@` as the only delimiter, so the `${...}` expressions of Quarkus stay as they are. Surefire and failsafe pass `quarkus.config.profile.parent=${bpms}` and `java.util.logging.manager=org.jboss.logmanager.LogManager` as system properties |
| Tests | every test class carries `@QuarkusTest` itself. The base class `WorkflowModuleTest` has none, and inheriting the annotation does not make the test a bean. A test runs in the Quarkus profile `test`, and the BPMS profile is its parent. A stand-in for a surrounding system is a bean with `@Alternative`, `@Priority(1)` and `@ApplicationScoped`, where Spring uses `@Primary` |
| Test of the workflow module | `vanillabp-quarkus-integration`, the adapter and a database in the test scope, and a test `application.yaml` with the database and nothing else |

Two things catch people on Quarkus:

- A Quarkus test resolves the application from the POM on its own. It knows the Maven profiles
  you name with `-P` on the command line, but not those in `MAVEN_ARGS` or those switched on in
  `settings.xml`. So name the BPMS profile and every profile your build needs with `-P`. The
  test `theTestRunsOnTheBpmsOfTheMavenBuild` of the blueprints tells you when the two disagree.
- If no database URL is configured, Quarkus Dev Services starts a database for you, for most
  databases in a container. Configure the database in the main and in the test
  `application.yaml`, as the blueprints do with H2, unless you want that.

VanillaBP checks the wiring between BPMN and code while Quarkus builds the application, not
only at startup. So most mistakes end `mvn verify` before any test runs. The message names the
fix.
