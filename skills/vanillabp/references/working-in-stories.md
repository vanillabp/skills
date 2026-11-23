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
1. [Driving sub-agents](#driving-sub-agents)
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
   The model decides which elements form one case, and a fixed rule decides the order, see
   [The same model gives the same stories](#the-same-model-gives-the-same-stories).
3. **Extra views and reports last.** They come after the process is complete.

Keep the stories small. A small story fits into the context of a weaker model, and its check
shows soon whether the model went the wrong way. A big story lets a model lose its way before
anything can be checked. The limits are in [Small stories](#small-stories).

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

In this example the user says that rejections are common, reminders are rarer, and withdrawals
are the rarest. So the rejection comes first, then the reminder, then the withdrawal.

Take the names of the blueprints from the catalogue, see
[Step 1](../SKILL.md#step-1-read-the-catalogue). It may have changed since this page was written.

### The same model gives the same stories

Two agents with the same model and the same answers of the user must get the same stories. Plans
can then be compared, and no model has to decide how to cut. So cut by these rules, and do not
change the cut by hand:

1. **The simplest path.** Walk from the start event to an end event. At each gateway take the flow
   on which everything goes well. If two flows fit, take the one which reaches an end event with
   fewer elements, and if that is a tie too, the default flow of the gateway. These elements are
   story 1.
2. **The groups of the special cases.** Each element off the simplest path belongs to one group.
   A group starts at each flow of a gateway that the simplest path does not take, at each boundary
   event and at each event subprocess. It holds every element you reach from there until the flow
   joins the simplest path again or ends. An element that two groups reach belongs to the group
   that comes first in the order of step 3.
3. **The order of the groups.** Order them by how often the case happens, the most frequent first.
   Take the frequency from the user or from the documentation of the model. A case which costs
   much when it goes wrong may come first, but only when the user says so. Where nobody knows the
   frequency, or two cases are equal, take the order of the elements in the BPMN file: the group whose first
   element stands higher in the file comes first. This command prints the line of each element:

   ```bash
   grep -nE 'id="(Gateway_RiskAcceptable|Event_ReminderAfterThreeDays|Event_Withdrawn)"' loan_approval.bpmn
   ```

4. **One group, one story.** Each group becomes one story, in that order, with the numbers after
   story 1. A group too big for one story is split as [Small stories](#small-stories) says.

Write the order and its source into the line `Order:` in the head of `plan/INDEX.md`, for example
"frequency from the user, 2026-11-19" or "nobody knows the frequency, order of the file". The next
agent then gets the same order.

### Small stories

A story changes at most about eight files, and six are better. It adds exactly one new
integration test, which walks through its case. It may adapt the tests of the stories before it.
Count each line of the list `Files` of the story file. Name files there, not folders, wherever you
know them. In story 1 the files of the base blueprint stand on one line, for example "the files of
module-process-driver (copy)", because they do not count.

When a story needs more than eight files, split it in two. The first story keeps the first
elements of its group in the order of the BPMN file, up to a point where the case already ends in
a valid state that a test can check. The second story takes the rest of the group. It gets the same
id with the letter `b`, for example `S05b`, in the line of `plan/INDEX.md` right after `S05`, with
its own story file, its own check command and its own new integration test. Split along the
business, never along the layers. "The model of the rejection" and "the code of the rejection" are
no stories. "The clerk rejects a loan request" and "The customer gets a letter of rejection" are.

## Planning a new application

Plan after you know the four answers of
[Before you write anything](../SKILL.md#before-you-write-anything), and after steps 1 to 4 of the
[Procedure](../SKILL.md#procedure). Then you know the blueprints and have read their `AGENTS.md`.

1. **Write down the structure.** Write the target layout as a tree into the section `Structure`
   of `plan/INDEX.md`, as [Step 4](../SKILL.md#step-4-read-the-agentsmd-of-each-chosen-blueprint)
   says. Story 1 builds all of it.
2. **Find the simplest path.** Walk through the model from the start event to an end event, and
   take at each gateway the flow on which everything goes well. Write down the elements on the
   way. This is story 1. Step 1 of
   [The same model gives the same stories](#the-same-model-gives-the-same-stories) says which flow
   to take when two fit.
3. **List the special cases.** Every element that is not on the simplest path belongs to a
   special case: the other flows of a gateway, a boundary event, a timer, a message that cancels.
   Group the elements and order the groups as
   [The same model gives the same stories](#the-same-model-gives-the-same-stories) says. Each
   group becomes one story, or two when it is too big, see [Small stories](#small-stories). If you
   took a process driver or a simulator, each of these stories extends them for its case.
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

Keep each story small: at most about eight files and one new integration test, as
[Small stories](#small-stories) says. If a story is bigger, split it along the business, not along
the layers. For example, "the clerk rejects a request" and "the clerk asks the customer for more
documents" are two stories, even if both start at the same gateway.

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
   this change on its own. In the story file, write the time into the line `Started:`, unless the
   agent that started you did that already.
2. Read the story file `plan/<id>.md`. Then read the sections it lists under `Read first`, and only
   those. Do not read whole wiki pages or whole blueprints, so your context stays small.
3. Work through the steps of the story file one by one. After each step, put a tick on it
   (`- [x]`) and save the file at once. The ticks are how others see your progress. If you stop in
   the middle of a step, write into `Notes` what is done and what comes next. The line `Time budget:` of the story file, 20 minutes unless it says
   otherwise, is a checkpoint. When the time is up and the check does not pass yet, write into
   `Notes` which step you are on and what holds you up, and tell the agent that started you, or the
   user. Then go on, unless they tell you otherwise.
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

## Driving sub-agents

Often one agent, the main agent, holds the plan and starts a sub-agent for each story. This section
is for the main agent. It steers and does not build: it writes no code, puts no tick on a step and
changes no test and no check. It reads `plan/INDEX.md`, the story files and `git log`, and it talks
to the sub-agent. A main agent that builds on its own fills its context with code, and then nobody
watches the sub-agent any more.

### The time budget

Every story file has the line `Time budget: 20 minutes`. Twenty minutes is the default for every
story. For a story that is clearly larger, for example story 1 with the whole Maven layout, the main
agent may raise the budget in the story file before the story starts. It writes the reason into
`Notes`. The budget is a checkpoint, not a limit. When it is used up, the main agent does not stop
the sub-agent. It looks at the ticks and the `Notes` and decides, as [At the
checkpoint](#at-the-checkpoint) says.

### Starting a story

1. Change the status of the story to `in-progress` and write the time into the line `Started:` of
   the story file.
2. Start one sub-agent. Tell it the root folder of the project and the story id, and that it works
   as [Working on one story](#working-on-one-story) says: a tick after each step, saved at once, and
   a stop at the end of the story.
3. Run one sub-agent at a time. The stories build on each other, and they all change the same
   model.

### Watching the progress

The ticks in the story file are the progress. Count them about every five minutes, and remember the
count:

```bash
grep -c '^- \[x\]' plan/S02.md
```

- There is a new tick since the last count: the sub-agent moves on. Look again later.
- There is no new tick since the last count: ask the sub-agent for its state and for what holds it
  up. A short answer is enough: the step it is on, the obstacle, and what it tries next.
- The sub-agent is stuck when one of these holds: there is still no new tick at the next count, or
  its answer shows that it goes round in circles, for example the same
  failing test with the same fix again.

### At the checkpoint

When the time budget is used up, the main agent decides. It does not stop the sub-agent for this
alone.

- The ticks still grow, and the steps left are few: let the sub-agent go on. Write a line into
  `Notes` with the time and a new checkpoint, for example ten minutes later.
- The ticks still grow, but many steps are left: let the sub-agent finish the step it is on. Then
  ask yourself whether the story is cut too big. If it is, split the rest as a story of its own.
- The ticks stopped, or the sub-agent goes round in circles: it is stuck. Go on as the next section
  says.

### When a sub-agent is stuck

1. Stop the sub-agent.
2. Write a note into `Notes` of the story file: the time, the step, the obstacle as the sub-agent
   named it, and what it tried.
3. Then do one of two things:
   - Split the story when it is too big for one go: the obstacle is a piece of work of its own, or
     the steps without a tick need more files than the story file names. Split it as
     [Small stories](#small-stories) says. The elements whose steps are done stay in the story. The
     other elements go into a new story with the same id and the letter `b`, for example `S02b`,
     in the line of `plan/INDEX.md` right after it, with its own story file, check command and
     integration test.
   - Start a new sub-agent on the same story with a hint when the obstacle is clear: the note, and
     one sentence about the way around, for example the section of the skill or the blueprint that
     answers it.
4. A story that gets stuck again after a hint is split. A story that cannot be split any more, for
   example one with a single element, goes back to `open` with a note when it gets stuck again.
   Write the obstacle into `plan/QUESTIONS.md`, as [Questions never block](#questions-never-block)
   says, and go on with the next story that does not depend on it.

### After a story

Before the next story, check the end of this one: the status in `plan/INDEX.md` is `done`, the last
commit names the story id, and the check command of the story passes. After story 1, also check
that `Notes` of story 1 holds the result of the comparison of the layout, step 9 of
[Working on one story](#working-on-one-story).

Running a check is steering, not building. If it fails, the story goes back to a sub-agent.

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
Order: rejection, reminder, withdrawal (frequency from the user, 2026-11-19)

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

The line `Order` names the special cases in the order of their stories, and where the order comes
from, see [The same model gives the same stories](#the-same-model-gives-the-same-stories). The line
`Read` says that you read the `AGENTS.md` of the organisation and the catalogue, as
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
Time budget: 20 minutes
Started:

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
whole application. One step is one file or one test, not "implement the process". The main agent
writes the time into `Started:` when the story starts. It raises `Time budget:` before the
start, with the reason in `Notes`, and writes a new checkpoint into `Notes` when it lets a
sub-agent go on after the budget, see [Driving sub-agents](#driving-sub-agents).

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
taken blueprint is in exactly one story, that every story names the `AGENTS.md` of the
organisation, that every story file has the lines `Time budget:` and `Started:`, and that no story
lists more than eight files.

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
  grep -q '^Time budget: ' "$story" \
    || { echo "plan: $story has no line 'Time budget:'"; failed=1; }
  grep -q '^Started:' "$story" \
    || { echo "plan: $story has no line 'Started:'"; failed=1; }
  files=$(awk '/^## /{inside=($0=="## Files")} inside && /^- /{n++} END{print n+0}' "$story")
  if [ "$files" -gt 8 ]; then
    echo "plan: $story lists $files files, split it as 'Small stories' says"
    failed=1
  fi
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
