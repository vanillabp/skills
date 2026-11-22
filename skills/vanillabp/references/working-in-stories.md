# Working in stories

*Part of the skill [`vanillabp`](../SKILL.md).*

A business process application is too big for one session. The context of an agent fills up, a
session ends, a weaker model takes over. So you never keep the plan in the chat. You keep it in the
project, in small stories, and you write down after every step what is done. Any agent can then
stop at any time, and the next one goes on from the files alone.

This holds for every job you cannot finish with one prompt: a new application, a new workflow
module, a new BPMN scenario in an existing application, an upgrade to a newer VanillaBP version. If
you expect more than one commit, or more than five changed files, make a plan. A small fix in one
or two files needs none.

**Contents:**

1. [The first step of every start: resume](#the-first-step-of-every-start-resume)
1. [The files of the plan](#the-files-of-the-plan)
1. [Stories are business slices](#stories-are-business-slices)
1. [Planning a new application](#planning-a-new-application)
1. [Planning an update](#planning-an-update)
1. [Working on one story](#working-on-one-story)
1. [Questions never block](#questions-never-block)
1. [Templates](#templates)

## The first step of every start: resume

Do this before anything else, every time you start, also when the user only says "go on". Run the
commands in the root folder of the project, where the root `pom.xml` is. For a new application that
does not exist yet, ask the user where it goes.

1. Look whether a plan is there:

   ```bash
   ls plan/INDEX.md plan/QUESTIONS.md
   ```

   - No `plan/INDEX.md`: plan the job. For a new application, see
     [Planning a new application](#planning-a-new-application). For a change to an existing one,
     see [Planning an update](#planning-an-update).
   - `plan/INDEX.md` is there: go on with step 2.
2. Look for changes nobody committed:

   ```bash
   git status --short
   git log --oneline -5
   ```

3. Look for a story that is in progress:

   ```bash
   grep -nE '\| *in-progress *\|' plan/INDEX.md
   ```

4. If there is one, run its check command from the column `Check command`:
   - The check passes: the work is done and only the end of the story is missing. Do steps 6 to 10
     of [Working on one story](#working-on-one-story).
   - The check fails: open the story file `plan/<id>.md`. Read its section `Notes` and the steps
     that have no tick yet. Go on with the first step without a tick.
5. If no story is in progress, but `git status` shows changes: these changes belong to no story.
   Do not throw them away. Run `git stash push -u -m "changes found when resuming"`, write a
   question about them into `plan/QUESTIONS.md`, and go on.
6. If no story is in progress, take the first line with the status `open`:

   ```bash
   grep -nE -m 1 '\| *open *\|' plan/INDEX.md
   ```

   Then work on it as [Working on one story](#working-on-one-story) says.
7. If there is no open story, the job is done. Tell the user, and name the questions in
   `plan/QUESTIONS.md` that still wait for an answer:

   ```bash
   grep -n '^Answer: open' plan/QUESTIONS.md
   ```

If two stories are in progress, take the first one and write a note about the second one into its
story file.

Tell the user in one or two sentences where you go on, for example: "S05 is in progress, its check
fails, I go on with step 3 of plan/S05.md."

## The files of the plan

All files of the plan are in the folder `plan/` at the root of the project. They are committed like
the code.

| File                       | What it holds                                                     |
|----------------------------|-------------------------------------------------------------------|
| `plan/INDEX.md`            | one line per story: id, title, status, check command              |
| `plan/<id>.md`             | one file per story: goal, what to read, files, steps, notes       |
| `plan/QUESTIONS.md`        | the questions that came up, each with the default you took        |
| `plan/checks/<id>.sh`      | only where a check command does not fit into the table, see below |
| `plan/checks/structure.sh` | the check of the layout, which every check command calls          |
| `plan/checks/plan-lint.sh` | the check of the plan itself, run before the plan is committed    |

The status of a story is one of three words: `open`, `in-progress` or `done`. `plan/INDEX.md` is the
only place that holds the status. Git holds the history: every story ends with one commit that
names its id.

A check command in the table must not contain a `|`, because a `|` breaks a Markdown table. If a
check needs one, or needs more than one line, write it into `plan/checks/<id>.sh` and put
`sh plan/checks/<id>.sh` into the table.

Number the stories `S01`, `S02` and so on, in the order you will work on them. A story id is a name
in the plan only. It never goes into the code, the tests or the texts of the application, see
[code-that-reads-like-prose.md](code-that-reads-like-prose.md#nothing-points-at-the-plan).

## Stories are business slices

A story is one thing the business can do. You build it through all layers at once: the BPMN
model, the workflow aggregate, the service code, the user task endpoints, the simulated outside
systems and the tests. A story is done when a user can do this thing in the running application,
and an integration test proves it.

Never build layer by layer. A plan such as "first the whole model, then all service tasks, then
all user tasks, then the tests" runs for a long time before anything works, and nobody can say
whether the process is right.

Cut the stories from the inside out:

1. **The core flow first.** Story 1 takes one case from the start to the end, on the simplest
   path. Choose the path with the fewest elements on which the process still does its job, mostly
   "everything goes well". It has no gateway, no deadline, no error and no cancelation.
2. **One special case per story.** Each next story adds one thing that can differ from the simple
   path: a rejection, a deadline, a cancelation, an escalation, an answer from a partner system.
   Take first the special cases that happen often or cost the most when they go wrong.
3. **Extra views and reports last.** They come after the process is complete.

A process driver and a simulator are no story of their own, and never the last one. They grow
with the process. Each story extends them for its own case: the driver gets the process state the
story reaches and the steps which lead there, and the simulator answers for every surrounding
system the story brings. This is part of the `Done when` of each story.

Story 1 also brings the technical base: the whole Maven layout of the section `Structure` in
`plan/INDEX.md`, and the persistence. It builds every module of that tree, also the ones which
stay nearly empty at first, such as a port with one method or a test driver with one step. The
layout is fixed in story 1, because every later story builds on it. Inside the modules, add
nothing that only a later story uses. The files the base blueprint brings do not count toward the
size of a story.

The model you get has every element of every path. Story 1 works on a copy that holds only the
elements of the simplest path. Each later story adds the elements of its special case from the
full model. So the model in the project always runs, and every element in it has its code and its
test. After the last special case, the model in the project has every element of the full model.

The blueprints still show how each element is built. Name a blueprint in the story that needs it
first, for example `bpmn-user-task` in the story that brings the first user task. A later story
that needs the same blueprint again reads the code of the earlier story instead.

Every blueprint the plan takes appears in exactly one story, under `Blueprints used first here`.
A taken blueprint which no story names is never built. The check
[plan/checks/plan-lint.sh](#a-plan-check-planchecksplan-lintsh) finds it.

### A worked example: a loan request

The BPMN model: a customer sends a loan request. A service task gets the credit rating. A clerk
assesses the risk in a user task. A gateway approves or rejects the request. A timer reminds the
clerk after three days. The customer can withdraw the request at any time. The user wants a
process driver, to bring a case to any point of the process while developing.

| Story | Business slice                                | Blueprints it names first                                                                           |
|-------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------|
| 1     | A loan request is approved, from start to end | `module-single`, `module-simulator`, `module-process-driver`, `bpmn-service-task`, `bpmn-user-task` |
| 2     | The clerk rejects a loan request              | `bpmn-gateways`                                                                                     |
| 3     | The clerk gets a reminder after three days    | `bpmn-timer`                                                                                        |
| 4     | The customer withdraws the request            | `bpmn-message-correlation`                                                                          |
| 5     | The process runs end to end                   | none                                                                                                |

Story 1 builds the whole layout of `module-process-driver`: the workflow module, the test driver,
the port `credit-rating` with its adapter and its `credit-rating-api` module, and the simulator.
It has the service task "Retrieve credit rating", the user task "Assess risk" and the end event
"Loan approved", and no gateway. The simulator answers the credit rating. The driver knows the
states "loan requested", "risk assessment opened" and "loan approved". Story 2 adds the gateway
and the flow to "Loan rejected", and the driver gets the state "loan rejected". Story 3 adds the
timer to the user task. Each of them adds one integration test that walks through its case, and
extends the driver and the simulator for it.

Take the names of the blueprints from the catalogue, see
[Step 1](../SKILL.md#step-1-read-the-catalogue). It may have changed since this page was written.

## Planning a new application

Plan after you know the four answers of
[Before you write anything](../SKILL.md#before-you-write-anything), and after steps 1 to 4 of the
[Procedure](../SKILL.md#procedure). Then you know the blueprints and have read their `AGENTS.md`.

1. **Write down the structure.** Write the target layout as a tree into the section `Structure`
   of `plan/INDEX.md`, as [Step 4](../SKILL.md#step-4-read-the-agentsmd-of-each-chosen-blueprint)
   says. Story 1 builds all of it.
2. **Find the simplest path.** Walk through the model from the start event to an end event, and
   take at each gateway the flow on which everything goes well. Write down the elements on the
   way. This is story 1.
3. **List the special cases.** Every element that is not on the simplest path belongs to a
   special case: the other flows of a gateway, a boundary event, a timer, a message that cancels.
   Group the elements by the business case they serve, for example "rejection". Each group
   becomes one story. Put them in the order of how often they happen. If you took a process
   driver or a simulator, each of these stories extends them for its case.
4. **Add the stories for extra views and reports**, if the user wants them.
5. **The last story.** "The process runs end to end": run the whole build, start the application,
   walk through the process with the logged URLs, and fix every message of the startup validation,
   as steps 6 and 7 of the [Procedure](../SKILL.md#procedure) say. Check that the model in the
   project now has every element of the model you got.
6. **More than one workflow module.** With `module-multi` or `module-interaction`, a business slice
   goes through every module its case touches. The story of the simplest path then brings the way
   the modules work together. Name the blueprint of the structure in that story.
7. **Link blueprints and stories.** Put every blueprint with `Taken: yes` into exactly one story,
   under `Blueprints used first here`. The `module-*` blueprints go into story 1.

Each story changes the model, the code and the tests together, and adds the attributes of
[Step 2](../SKILL.md#step-2-parse-the-bpmn) to the elements it brings. A service task or user task
whose business code is more than a call gets that code in the story that brings the task.

Every story file lists under `Read first` the `AGENTS.md` of the organisation, at least its list
"Never do this", and the `AGENTS.md` of every taken blueprint the story touches. An agent which
works on a story knows only its brief and the story file. So the rules have to be named there.

Keep each story small. A story changes at most about ten files and adds or adapts one or two
tests. If a story is bigger, split it along the business, not along the layers. For example, "the
clerk rejects a request" and "the clerk asks the customer for more documents" are two stories,
even if both start at the same gateway.

A check must fail before the story is done and pass after it. A build that passes from the start
proves nothing. So a check first runs the structure check, then tests that the new test file
exists, then runs the build of the module:

```bash
sh plan/checks/structure.sh && test -f loan-approval/workflow-module/src/test/java/blueprint/workflowmodule/loanapproval/RejectLoanIT.java && mvn -B -pl loan-approval/workflow-module -am verify -Pcamunda7
```

A green build does not prove the structure. A workflow module which imports a generated client
compiles fine. So write `plan/checks/structure.sh` now, from the tree in `Structure`, see the
[template](#a-structure-check-planchecksstructuresh). It fails until story 1 has built the layout,
and the check command of every story calls it. A story which adds a module to the tree adds its
line to the script.

Write `plan/INDEX.md`, one story file per line, an empty `plan/QUESTIONS.md`,
`plan/checks/structure.sh` and `plan/checks/plan-lint.sh`. Use the [Templates](#templates). Run the plan check. Commit the plan
only when it passes:

```bash
sh plan/checks/plan-lint.sh
git add plan/
git commit -m "Plan the application in stories"
```

## Planning an update

An update changes an application that exists. The planning first reads the project and writes down
what it found. Only then does it write stories.

1. Write down what the project is. Put the result into the head of `plan/INDEX.md`:

   ```bash
   grep -rn --include=pom.xml -A2 'io.vanillabp' . | grep -E 'artifactId|version'
   grep -rln --include=*.java '@WorkflowService' .
   find . -name '*.bpmn' -not -path '*/target/*'
   grep -rn --include=*.yaml --include=*.yml --include=*.properties 'vanillabp' . | grep -v /target/
   ```

   This gives the VanillaBP version and the adapters, the workflow modules, the BPMN models and the
   configuration.
   Write the Maven modules of the project as a tree into the section `Structure`, and
   `plan/checks/structure.sh` from it. An update keeps the layout the project has, unless a story
   changes it on purpose.
2. Read what the update needs:
   - **A newer VanillaBP 2 version.** The release notes of each VanillaBP repository the project
     uses, from the version it has to the one it gets, and the wiki pages they link.
   - **VanillaBP 1 to 2.** The skill `vanillabp-upgrade-v1-to-v2`. Its survey in step 0 is the
     reading. Its list of findings becomes the stories.
   - **A new BPMN scenario or a new workflow module.** Steps 1 to 4 of the
     [Procedure](../SKILL.md#procedure), as for a new application.
3. Cut the work into stories. Where the business changes, cut business slices. Where only the
   technique changes, cut technical stories. Use this order:
   1. One story that raises the versions in the `pom.xml` files and nothing else. Check:
      `mvn -B test-compile`. It may fail, then the next stories fix it. A plain raise of the
      version is a technical story. It changes nothing a user notices.
   2. One story per workflow module that adapts its code until it compiles and its tests pass.
      Check: `mvn -B -pl <module> -am verify -P<profiles>`.
   3. One story per configuration change, for example a moved key. Check: the build of the module,
      plus a `grep` that the old key is gone, for example
      `! grep -rn 'vanillabp.default-adapter' --include=*.yaml .`.
   4. Business slices for every change of the business, as in
      [Stories are business slices](#stories-are-business-slices). A new special case is one
      story. A new process starts with the story of its simplest path.
   5. The last story: the whole build and one start of the application with every profile it
      knows. Fix each message of the startup validation, as step 7 of the
      [Procedure](../SKILL.md#procedure) says.
4. Write the plan and commit it, as for a new application.

## Working on one story

These are the steps for one story. Do them in this order, and do not skip one.

1. In `plan/INDEX.md`, change the status of the story from `open` to `in-progress`. Do not commit
   this change on its own.
2. Read the story file `plan/<id>.md`. Then read the sections it lists under `Read first`, and only
   those. Do not read whole wiki pages or whole blueprints, so your context stays small.
3. Work through the steps of the story file one by one. After each step, put a tick on it
   (`- [x]`) and save the file. If you stop in the middle of a step, write into `Notes` what is done
   and what comes next.
4. When something comes up that you have to decide and cannot find in the skill or the blueprints,
   do what [Questions never block](#questions-never-block) says.
5. Run the check command of the story. If it fails, fix the code and run it again. Never change a
   test or a check only to make it pass. A message of the startup validation names the fix, see
   [Step 7](../SKILL.md#step-7-fix-what-startup-validation-reports).
6. Read your own change once against [code-that-reads-like-prose.md](code-that-reads-like-prose.md)
   and run the checks there.
7. In `plan/INDEX.md`, change the status from `in-progress` to `done`.
8. Commit the story, in one commit, with the story id in the message:

   ```bash
   git add -A
   git commit -m "S02 The clerk rejects a loan request"
   ```

9. After story 1 only: compare the layout with the reference before any other story starts.
   Story 1 sets the layout for all later ones, so this is the cheapest place to catch a mistake.
   Put the module tree of the project next to the tree of the base blueprint and next to the
   section `Structure` of `plan/INDEX.md`:

   ```bash
   find . -name pom.xml -not -path '*/target/*' -not -path './plan/*' | sort
   ```

   Look at the dependencies as well. The workflow module names only ports, never an adapter or a
   `client` module. The application names the adapters. If something differs, fix it in a new
   commit before you go on, and write what you found into `Notes` of story 1.
10. Take the next story: go back to step 6 of
    [The first step of every start](#the-first-step-of-every-start-resume).

After each story you may stop. Stop also when the user asks you to, or when your context is nearly
full. The files hold everything the next agent needs.

## Questions never block

When you do not know something, you do not wait for an answer. You write the question down with a
default, take the default and go on.

1. Pick the default. Take the default the skill or the blueprint names. If they name none, take the
   choice that is easiest to change later and that commits to nothing. For a gateway, that is the
   flow which does not spend money or order anything, as
   [Step 2](../SKILL.md#step-2-parse-the-bpmn) says.
2. Add an entry to `plan/QUESTIONS.md`, see the [template](#an-entry-of-planquestionsmd). Give it the
   next free number, `ASK-001`, `ASK-002` and so on.
3. Write one sentence into `Notes` of the story file: which question, which default.
4. Go on with the default.

When the user answers later, write the answer into the entry. If it differs from the default, add a
new story at the end of `plan/INDEX.md` that changes the default into the answer. Its title names
the question, for example "Apply the answer to ASK-003".

A finished plan stays in the project. When a new job starts and `plan/INDEX.md` has no open story,
move the old plan away first:

```bash
mkdir -p plan/archive/2026-11-19
git mv plan/INDEX.md plan/QUESTIONS.md plan/S*.md plan/archive/2026-11-19/
git commit -m "Archive the finished plan"
```

Use the date of the day. Move `plan/checks` as well, if it is there. The ids of the new plan start
at `S01` again.

## Templates

### plan/INDEX.md

```markdown
# Plan

Job: loan approval application, Spring Boot, Camunda 7, JPA
Started: 2026-11-19
Profiles: camunda7
Read: AGENTS.md of the organisation and blueprints.yaml (2026-11-19), AGENTS.md of every taken blueprint

## Structure

Taken from module-process-driver, the deepest taken blueprint of the chain of `base` fields.

    application/                             depends on the workflow module and the adapters
    loan-approval/gui-api/client/            the GUI API, used by the test driver
    loan-approval/gui-api/server/            the GUI API, implemented by the workflow module
    loan-approval/workflow-module/           depends on the ports only
    loan-approval/test-driver/               uses the client of the GUI API and of the simulator
    external-systems/credit-rating/port/
    external-systems/credit-rating/credit-rating-api/client/
    external-systems/credit-rating/credit-rating-api/server/
    external-systems/credit-rating/crediscore-adapter/
    simulator/api/client/
    simulator/api/server/
    simulator/application/

## Blueprints

| Blueprint                  | Taken | Reason                                                      |
|----------------------------|-------|-------------------------------------------------------------|
| module-single              | yes   | one workflow module, the base of module-simulator           |
| module-multi               | no    | `module-structure`: module-single taken                     |
| module-simulator           | yes   | the base of module-process-driver                           |
| module-process-driver      | yes   | the user wants to play through the process while developing |
| persistence-mongodb        | no    | `aggregate-storage`: JPA of module-single is enough         |
| bpmn-timer                 | yes   | the model has a boundary timer on "Assess risk"             |
| bpmn-signals               | no    | the model has no signal                                     |
| cockpit-user-task-frontend | no    | the job has no user interface of its own                    |

## Stories

| Id  | Title                                         | Status      | Check command                                                                                                                            |
|-----|-----------------------------------------------|-------------|------------------------------------------------------------------------------------------------------------------------------------------|
| S01 | A loan request is approved, from start to end | done        | `sh plan/checks/structure.sh && test -f loan-approval/workflow-module/src/test/java/.../ApproveLoanIT.java && mvn -B verify -Pcamunda7`  |
| S02 | The clerk rejects a loan request              | in-progress | `sh plan/checks/structure.sh && test -f loan-approval/workflow-module/src/test/java/.../RejectLoanIT.java && mvn -B verify -Pcamunda7`   |
| S03 | The clerk gets a reminder after three days    | open        | `sh plan/checks/S03.sh`                                                                                                                  |
| S04 | The customer withdraws the request            | open        | `sh plan/checks/structure.sh && test -f loan-approval/workflow-module/src/test/java/.../WithdrawLoanIT.java && mvn -B verify -Pcamunda7` |
| S05 | The process runs end to end                   | open        | `sh plan/checks/structure.sh && mvn -B verify -Pcamunda7`                                                                                |
```

The line `Read` says that you read the `AGENTS.md` of the organisation and the catalogue, as
[Step 1](../SKILL.md#step-1-read-the-catalogue) asks, and when. The section `Structure` is the
target layout of [Step 4](../SKILL.md#step-4-read-the-agentsmd-of-each-chosen-blueprint), with the
names of your project. The adapter is named after the real system, here an invented `crediscore`.
The section `Blueprints` lists every entry of the catalogue, taken or left out, with the reason, as
[Step 3](../SKILL.md#step-3-pick-the-blueprints) says. The example shows only some rows. Write the
full path in place of `...`. Each title says what a user can do when the story is done.
For an update, the head also holds what step 1 of [Planning an update](#planning-an-update) found:
the VanillaBP version before and after, the adapters and the workflow modules.

### A story file, plan/S02.md

```markdown
# S02 The clerk rejects a loan request

Workflow module: loan-approval
Blueprints used first here: bpmn-gateways
Depends on: S01

## Goal

In the user task "Assess risk", the clerk can say that the risk is too high. Then the process
ends at "Loan rejected" and the customer gets a letter of rejection.

## Done when

The test RejectLoanIT starts a loan request, completes "Assess risk" with "risk too high", and
sees the process end at "Loan rejected". The test ApproveLoanIT of S01 still passes. The process
driver brings a case to the new state "loan rejected".

## Read first

- AGENTS.md of the organisation, section "Never do this"
- AGENTS.md of the blueprint bpmn-gateways, section "Adding this blueprint to an existing project"
- AGENTS.md of the blueprint module-process-driver, the part about process states and steps
- plan/S01.md, section "Notes"
- vanillabp skill, SKILL.md, section "Step 2: parse the BPMN", the part about gateways
- loan-approval/workflow-module/src/main/java/blueprint/workflowmodule/loanapproval/assessrisk/

## Files

- loan-approval/workflow-module/src/main/resources/loan-approval/processes/camunda7/loan_approval.bpmn (change)
- loan-approval/workflow-module/src/main/java/blueprint/workflowmodule/loanapproval/assessrisk/ (change)
- loan-approval/workflow-module/src/main/java/blueprint/workflowmodule/loanapproval/LoanApprovalService.java (change)
- loan-approval/workflow-module/src/test/java/blueprint/workflowmodule/loanapproval/RejectLoanIT.java (new)
- loan-approval/test-driver/src/main/java/blueprint/testdriver/loanapproval/ (change)

## Steps

- [ ] 1. Copy ApproveLoanIT.java to RejectLoanIT.java. Change it so that the clerk answers "risk
      too high" and the test expects the end event "Loan rejected". Run the check. It fails.
- [ ] 2. Add the field for the answer of the clerk to the workflow aggregate.
- [ ] 3. Add the gateway "Risk acceptable?", the end event "Loan rejected" and the service task
      "Send rejection" from the full model to the model of the project. Give the gateway its
      conditions and its default flow.
- [ ] 4. Add the endpoint parameter for the answer to the user task "Assess risk" in the OpenAPI
      file of the GUI API.
- [ ] 5. Write the method of the service task "Send rejection" in LoanApprovalService.java.
- [ ] 6. Add the process state "loan rejected" and the step which leads there to the test driver.
- [ ] 7. Run the check. It passes.

## Notes

(Written while working: what is done, what was tried, what is left.)
```

Write the steps so concrete that a weaker model can do each of them without thinking about the
whole application. One step is one file or one test, not "implement the process".

### A check script, plan/checks/S03.sh

```sh
#!/bin/sh
set -e
sh plan/checks/structure.sh
test -f loan-approval/workflow-module/src/test/java/blueprint/workflowmodule/loanapproval/ReminderIT.java
grep -q 'timeDuration' loan-approval/workflow-module/src/main/resources/loan-approval/processes/camunda7/loan_approval.bpmn
mvn -B verify -Pcamunda7
```

### A structure check, plan/checks/structure.sh

Write it from the section `Structure`. It checks what a green build does not: that every module
of the tree is there, and that the dependencies point the right way.

```sh
#!/bin/sh
# The layout of plan/INDEX.md, section "Structure". A green build does not prove it.
set -e
fail() { echo "structure: $1"; exit 1; }

for module in \
    application \
    loan-approval/gui-api/client \
    loan-approval/gui-api/server \
    loan-approval/workflow-module \
    loan-approval/test-driver \
    external-systems/credit-rating/port \
    external-systems/credit-rating/credit-rating-api/client \
    external-systems/credit-rating/credit-rating-api/server \
    external-systems/credit-rating/crediscore-adapter \
    simulator/api/client \
    simulator/api/server \
    simulator/application; do
  test -f "$module/pom.xml" || fail "the module $module is missing"
done

# The workflow module knows the ports only: no adapter and no generated client in its code.
! grep -rnE 'import [a-z0-9_.]+\.(api\.client|crediscore)\.' loan-approval/workflow-module/src/main \
  || fail "the workflow module uses an adapter or a generated client"

# Nobody writes a REST client by hand.
! grep -rnE 'RestTemplate|WebClient|@HttpExchange|@GetExchange|@PostExchange' \
    --include=*.java application loan-approval external-systems simulator \
  || fail "a REST client is written by hand, use the generated one"

# The application, not the workflow module, pulls in the adapter.
grep -q '<artifactId>credit-rating-crediscore-adapter</artifactId>' application/pom.xml \
  || fail "the application does not depend on the adapter of the port credit-rating"
```

### A plan check, plan/checks/plan-lint.sh

Run it before you commit the plan, and again whenever you change the plan. It checks that every
taken blueprint is in exactly one story, and that every story names the `AGENTS.md` of the
organisation.

```sh
#!/bin/sh
# Every taken blueprint is first used in exactly one story.
failed=0

taken=$(grep -E '^\| *[a-z0-9-]+ *\| *yes *\|' plan/INDEX.md | cut -d'|' -f2 | tr -d ' ')
for blueprint in $taken; do
  count=$(grep -h '^Blueprints used first here:' plan/S*.md \
    | sed 's/^[^:]*://' | tr ',' '\n' | tr -d ' `' | grep -cx "$blueprint")
  if [ "$count" -ne 1 ]; then
    echo "plan: $blueprint is taken, but $count stories name it under 'Blueprints used first here'"
    failed=1
  fi
done

grep -q '^Read: ' plan/INDEX.md \
  || { echo "plan: plan/INDEX.md does not say which AGENTS.md files were read"; failed=1; }
grep -q '^## Structure' plan/INDEX.md \
  || { echo "plan: plan/INDEX.md has no section Structure"; failed=1; }
for story in plan/S*.md; do
  grep -q 'AGENTS.md of the organisation' "$story" \
    || { echo "plan: $story does not name the AGENTS.md of the organisation under Read first"; failed=1; }
done

exit $failed
```

### An entry of plan/QUESTIONS.md

```markdown
## ASK-003 Which flow of the gateway "Risk acceptable?" is the default?

Story: S02
Date: 2026-11-19
Default taken: "no", the flow which rejects the loan request.
Why this default: it spends no money and commits to nothing.
Answer: open
```

`Answer: open` stays until the user answers. Then it holds the answer, for example
`Answer: yes, accept is the default (2026-11-21)`.
