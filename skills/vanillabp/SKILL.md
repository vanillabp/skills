---
name: vanillabp
description: Build or extend a business process application with VanillaBP. Turns a BPMN model into a workflow module, grafts BPMN scenarios such as user tasks, message correlation, timers, boundary events, call activities or multi-instance onto an existing application, and picks persistence and BPMS adapter configuration. Use whenever a project uses VanillaBP, whenever a BPMN model has to be implemented in Java, and whenever the terms workflow module, workflow aggregate, @WorkflowService, @WorkflowTask or ProcessService come up.
license: Apache-2.0
metadata:
  author: vanillabp
  homepage: https://www.vanillabp.io
---

# Building applications with VanillaBP

VanillaBP applies hexagonal architecture to business process applications. A BPMN (Business
Process Model and Notation) model drives the process. Your code implements the tasks in it.
None of that code knows which business process engine (BPMS) runs it. Choosing the BPMS is a
Maven profile, not a code change. That is why nothing you write here may refer to an engine
API.

Two terms carry the whole design:

A *workflow module* bundles the BPMN models of one use case together with the code which
implements them. It is normally a JAR which an application pulls in. The application decides
which BPMS adapter is on the classpath.

A *workflow aggregate* is one stored entity per workflow instance. It holds every piece of
state the process needs. There are no process variables.

## Start here: resume

Every start begins with the same step, also the first one and also when the user only says "go
on": resume. Run the commands of
[The first step of every start](references/working-in-stories.md#the-first-step-of-every-start-resume)
in the root folder of the project. They tell you whether to plan the job or to go on with a story.

A job that needs more than one prompt never lives only in the chat. This holds for a new
application, a new workflow module, a new BPMN scenario and an upgrade to a newer VanillaBP version
alike. The plan lives in the project, in `plan/INDEX.md` and one file per story. Each story is a
small business slice: one thing the business can do, built through the model, the code and the
tests at once. The first story runs one case from start to end on the simplest path, and each
next story adds one special case. A story changes at most about eight files and adds one new
integration test. Fixed rules cut and order the stories, so the same model gives the same plan.
Each story has a check command that shows when it is done. Any
agent, also a weaker model with a small context, can stop after any step and the next one goes on
from the files.
[references/working-in-stories.md](references/working-in-stories.md) has the procedure and the
templates.

The code you write explains itself: names read like small sentences, comments say why, and the
English is plain. [references/code-that-reads-like-prose.md](references/code-that-reads-like-prose.md)
has the rules and the checks you run before each commit.

## Before you write anything

Find out these four. Ask the user where the project does not answer them:

1. The platform, Spring Boot or Quarkus. Everything after this differs, and the two are never
   mixed in one project. On Quarkus, read
   [What is different on Quarkus](references/project-structure.md#what-is-different-on-quarkus)
   before you write the first class.
2. The BPMS: Camunda 7, Camunda 8 or the bpm-crafters Process-Engine-API.
3. Whether this is a new application or an existing one to extend.
4. The BPMN model. Without a model there is nothing to implement. So ask for the file instead
   of inventing a process.

## Procedure

Steps 1 to 4 are the planning. After step 4, write the plan as
[Planning a new application](references/working-in-stories.md#planning-a-new-application) says.
Steps 5 to 7 then run once per story, each story with its own check command. If you hand the
stories to sub-agents, you steer and do not build: give each story a time budget, watch its ticks
and decide when a sub-agent is stuck, as
[Driving sub-agents](references/working-in-stories.md#driving-sub-agents) says.

### Step 1: read the catalogue

The blueprints are the reference implementations: one small application you can run, per
aspect. A catalogue for machines lists them,
[blueprints.yaml](https://github.com/vanillabp-blueprints/.github/blob/main/blueprints.yaml).
Fetch both files in their raw form:

- `https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/blueprints.yaml`
- `https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/AGENTS.md`

The catalogue is the only list of blueprints. It grows as blueprints are added, and this skill
keeps no copy of it, so a new blueprint is used the day it is delivered. Where the catalogue
disagrees with anything in this skill, the catalogue wins. If you cannot fetch it, do not pick
blueprints from memory: ask the user for the file or for network access.
[references/catalogue.md](references/catalogue.md) explains what its fields mean.

Read the `AGENTS.md` in full. It holds the procedure, the reference structure and the list
"Never do this", which holds for every blueprint. This step is a gate: the head of
`plan/INDEX.md` says that you read both files, and every story lists the `AGENTS.md` under
`Read first`. See the [template](references/working-in-stories.md#planindexmd).

### Step 2: parse the BPMN

Collect the element types the model uses (`bpmn:UserTask`, `bpmn:BoundaryEvent`,
`bpmn:MessageEventDefinition` and so on), together with the attributes set on them. This list
selects the blueprints.

A model you get from a business analyst often has no engine attributes at all. It names the
tasks and the flows, but nothing in it names the task definition of a service task or the form
of a user task, and its gateways have no conditions and no default flow. Such a model does not
run yet. You add what is missing in step 5, in the copy of the model below
`<module-id>/processes/<adapter-id>/`. A business deadline written as a literal, such as `P3D`
on a timer, becomes an expression which reads the workflow aggregate, as the blueprint
`bpmn-timer` shows. The `bpmn-*` blueprints show each attribute in their models. The wiki of
each BPMS adapter lists in one table what VanillaBP expects of the model, for example
[The BPMN model for Camunda 7](https://github.com/vanillabp/camunda7-adapter/wiki/Configuration#the-bpmn-model-for-camunda-7)
and [The BPMN model for Camunda 8](https://github.com/vanillabp/camunda8-adapter/wiki/Configuration#the-bpmn-model-for-camunda-8).

Which flow of a gateway is the default is a business decision, not a technical one. Ask the
user. If nobody can answer, pick the side which does not spend money or commit to anything, for
example "reject" rather than "order". Then say in your answer which default you picked and why.

### Step 3: pick the blueprints

Go through EVERY entry of the catalogue, not only the ones whose name looks right, and decide
for each one: take it or leave it out. Leave an entry out only for a reason you can name:

- the model or the user's request does not need it. For a `bpmn-*` entry, compare `covers.bpmn`
  with the element types of step 2. Some entries cover no element type at all, for example
  versioning or a driver which plays through the process during development. Decide those from
  what the user wants to build, not from the model;
- its `platforms.<platform>.status` is not `available`. Somebody works on a `planned` entry. A
  `not-applicable` entry names the reason why that platform cannot have it. Do not port it
  yourself. For a frontend, an entry whose id ends in `-frontend`, `<platform>` is the UI
  framework of the job (`vue`, `react` or `angular`), not Spring Boot or Quarkus. Its
  `not-applicable` for `springboot` and `quarkus` is expected and no reason to leave it out.
  A frontend entry whose key for your UI framework is missing or not `available` is no reason
  either, if the job needs what it shows. Take it as a pattern: read its `AGENTS.md` and its code
  in the UI framework it has, and build the same thing in yours. Write `yes, as a pattern` into
  the plan. Leave such an entry out only if the job does not need what it shows;
- you took another entry of the same `choose_one` group (see below);
- the user said no.

Entries with the same `choose_one` value are alternatives: take at most one of each group. From
`module-structure` take exactly one, starting at `module-single`. Taking none of another group
means the default which `module-single` already shows, for example JPA with a repository when
you take nothing from `aggregate-storage`. The schema of the catalogue says per group what
taking none means. An entry without `choose_one` is an addition. Take it on top of the others
when it applies.

If you cannot tell whether an entry applies, ask the user. If nobody can answer, take the
cheaper choice and say which one you took.

Write the result as a table into the plan, section `Blueprints` of `plan/INDEX.md`: every entry
of the catalogue, taken or left out, and the reason. The user sees what was left out and can object, and a blueprint added
to the catalogue later cannot slip through unnoticed.

### Step 4: read the AGENTS.md of each chosen blueprint

Every blueprint repository has one. The catalogue links it as `platforms.<platform>.agents_md`.
It names the placeholders, the files which carry the aspect, and the steps to add it to an
existing project.

Then write down the target layout of the project. The taken `module-*` blueprints decide the
Maven modules: ports and adapters in modules of their own, an `<name>-api` module per API, the
simulator, the test driver, one module per workflow module. Take the layout of the taken
blueprint which is deepest in the chain of `base` fields, and write it as a tree into the section
`Structure` of `plan/INDEX.md`.
[The Maven layout comes from the `module-*` blueprints](references/project-structure.md#the-maven-layout-comes-from-the-module--blueprints)
shows the tree of `module-process-driver`.

### Step 5: apply the deltas

The blueprint which gives the layout is the base. That is `module-single`, unless you took a
`module-*` blueprint which builds on it, such as `module-simulator` or `module-process-driver`.
Every `bpmn-*` blueprint is a delta on top of the base. They all have the same structure. So
several deltas combine into one application.

For a new application, clone the delivered repository of the base blueprint for your platform
(`https://github.com/vanillabp-blueprints/<blueprint-id>-<platform>`). Take every delta from the
twin of the same platform. Keep every Maven module of the base, also the ones your first story
does not fill yet. A Spring Boot twin has an `Application` class, an auto-configuration
and a `TestApplication`, and a Quarkus project needs none of them. Replace all the
placeholders its AGENTS.md lists, consistently. For an existing application, follow the section
"Adding this blueprint to an existing project" in each blueprint's AGENTS.md. Keep the
structure the project already has.

[references/project-structure.md](references/project-structure.md) has the structure every
blueprint uses, the placeholders and the layout of the configuration. Read it before you write
the first class.

Every API is API-first. An OpenAPI file describes it, and the build generates the code on both
sides, in Java and in the TypeScript of a user interface. Nobody writes a REST client by hand.
Each API with a Java caller gets a module with a `client` and a `server` module:
`<use-case>/gui-api` for the GUI API of a workflow module, `<name>-api` for any other API.
Client and server are seen from the caller: the test driver uses the client of the GUI API, and
the workflow module implements its server. For a surrounding system, the simulator implements the
server, and the adapter of the port uses the client. A port is named after what it is for, such
as `e-signature`, and its adapter after the system, such as `sign-now`.
[Every API is an OpenAPI file](references/project-structure.md#every-api-is-an-openapi-file-and-all-code-is-generated-from-it)
has the details, callbacks included. [references/rules.md](references/rules.md) has the rules which hold for every
blueprint. If you break one of them, a VanillaBP application stops working without a word.

A workflow module with parts for people in a Business Cockpit, such as the form of a user task,
the page of a case or a button in the header, is delivered with a dev shell. The dev shell is a
small web application which stands in for the cockpit while somebody works on these parts. It
draws them from their source files, so a change shows while you type, and it needs no cockpit,
no MongoDB and no BPMS. Behind it runs the dev shell simulator of the cockpit, or the simulator of
the module which extends it. Build it for each user interface project of the module, in its UI
framework, with the dev shell package of the cockpit (`<scope>/bc-dev-shell-<framework>`) and
its entry point, for example `bootstrapDevShell` in React. One npm script starts it on the dev
server of the framework, with the proxy the package brings. It shows every part of the module.
A BPMN diagram, on the page of a case, in a dev shell or as a picture in a build, is drawn with
[bpmn-js](https://bpmn.io/toolkit/bpmn-js/). Never draw it yourself from the diagram part of the
model. In a user interface use its Viewer or NavigatedViewer and mark the current and the passed
steps with markers or overlays. Take the package from npmjs, also when the npm registry of the
cockpit does not carry it, and load it only on the page which shows the diagram. A build renders
a picture with [bpmn-to-image](https://github.com/bpmn-io/bpmn-to-image). The bpmn.io logo stays
visible, its license asks for it.

If the project has a process driver, the dev shell also gets a page "Process driver", see
[The page "Process driver"](references/project-structure.md#the-page-process-driver-of-the-dev-shell).
What the dev shell must do is in the skill `business-cockpit`, section
[The dev shell is part of the delivery](https://github.com/vanillabp/skills/blob/main/skills/business-cockpit/SKILL.md#the-dev-shell-is-part-of-the-delivery).
A module without a dev shell is not done.

Such a module names its access groups in `getAccessibleToGroups` of its
`WorkflowModuleDetailsProvider`. A user who is in none of them never loads a file of the module,
so they cannot open the form of their own task. So every candidate group of a user task of the
module is an access group, and so is every group which may see a case of the module. A feature
role, for example the right to use a button in the header, implies an access group. It never
replaces one. The blueprint
[`cockpit-header-extension-backend`](https://github.com/vanillabp-blueprints/blueprints/tree/main/blueprints-cockpit/cockpit-header-extension-backend)
keeps the two apart.

### Step 6: verify by running it

```bash
mvn verify
```

Name every Maven profile the build needs with `-P` on the command line. A Quarkus test reads
them only from there.

Every blueprint ships an integration test which plays through its aspect. Keep that test and
adapt it to the use case. Generated code which never ran is a guess.

### Step 7: fix what startup validation reports

VanillaBP checks the wiring between BPMN and code while the application boots. On Quarkus it
already checks while the application builds. Its messages name the fix. Never work around such
a message. It describes a real inconsistency. Follow the message step by step until the
configuration is complete.

If you cannot place a message, look in [references/symptoms.md](references/symptoms.md). It
leads from what you see to the section which explains it.

Steps 6 and 7 are the only reason to trust the result. Do not skip them.

## Operating the result

The path of every endpoint of a workflow module starts with the module id, then `api`:
`/loan-approval/api/...`, never `/api/loan-approval/...`.

Every endpoint of a blueprint is a GET request, also the ones which change something: the
start, the completion of a user task and the callback which a surrounding system calls. So you
can walk through a process in a browser with no tooling at all. Keep that property in what you
generate, also in an application without a user interface. At every wait state, log one
complete, clickable URL for each possible next step.

```
Accept -> http://localhost:8080/loan-approval/api/{id}/assess-risk/{taskId}?riskIsAcceptable=true
Deny   -> http://localhost:8080/loan-approval/api/{id}/assess-risk/{taskId}?riskIsAcceptable=false
```

Those logged URLs are also the cheapest way for you to see which state a workflow reached.

The one exception are the endpoints which a user interface calls. The user interface calls
them, not the user, so they may use other methods. A form loads its data with GET, saves a draft
with PUT and completes its user task with POST. Only the endpoints of the user interface do
this. The start, the views and the callback of a surrounding system stay GET, also in an
application which has a user interface. If the user interface is not part of the application,
for example because it is built in a project of its own, log the `curl` command which stands in
for it. While somebody works on the user interface, the dev shell calls these endpoints through
the proxy of its dev server, below `/wm/<module-id>/`, the same paths as in the cockpit.

## When something is not covered

Blueprints cover only what the SPI (service provider interface) and the platform integrations
provide, on purpose. If a task cannot be solved without details of a BPMS, say so instead of
inventing configuration. [references/documentation-map.md](references/documentation-map.md)
says where each topic is documented.

## When something goes wrong

Start at [references/symptoms.md](references/symptoms.md). It lists the messages and exceptions
which come up around VanillaBP. It says what each one is and links the section which answers
it. Search it with the words you see before you search the wikis. Add a row whenever you find
an answer which was not there.

Where the application reports into a VanillaBP Business Cockpit, search
[references/business-cockpit-symptoms.md](references/business-cockpit-symptoms.md) as well. It
is the same kind of list, written for the cockpit server and for the adapters which report into
it. For work on the cockpit side itself, not only reading an error from it, use the skill
`business-cockpit`.

Upgrading an application from VanillaBP 1 is a different job with its own skill,
`vanillabp-upgrade-v1-to-v2`. Plan it in stories all the same, as
[Planning an update](references/working-in-stories.md#planning-an-update) says.
