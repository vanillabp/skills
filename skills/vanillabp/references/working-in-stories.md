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
   - The check passes: the work is done and only the end of the story is missing. Do steps 6 to 9
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

| File                  | What it holds                                                        |
|-----------------------|----------------------------------------------------------------------|
| `plan/INDEX.md`       | one line per story: id, title, status, check command                 |
| `plan/<id>.md`        | one file per story: goal, what to read, files, steps, notes          |
| `plan/QUESTIONS.md`   | the questions that came up, each with the default you took           |
| `plan/checks/<id>.sh` | only where a check command does not fit into the table, see below    |

The status of a story is one of three words: `open`, `in-progress` or `done`. `plan/INDEX.md` is the
only place that holds the status. Git holds the history: every story ends with one commit that
names its id.

A check command in the table must not contain a `|`, because a `|` breaks a Markdown table. If a
check needs one, or needs more than one line, write it into `plan/checks/<id>.sh` and put
`sh plan/checks/<id>.sh` into the table.

Number the stories `S01`, `S02` and so on, in the order you will work on them. A story id is a name
in the plan only. It never goes into the code, the tests or the texts of the application, see
[code-that-reads-like-prose.md](code-that-reads-like-prose.md#nothing-points-at-the-plan).

## Planning a new application

Plan after you know the four answers of
[Before you write anything](../SKILL.md#before-you-write-anything), and after steps 1 to 4 of the
[Procedure](../SKILL.md#procedure). Then you know the blueprints and have read their `AGENTS.md`.
The plan follows the blueprints: one story per workflow module and blueprint.

1. **The base.** One story: clone the base blueprint of the chosen `module-*` structure for your
   platform, replace every placeholder its `AGENTS.md` lists, and make its own integration test
   pass. Check: `mvn -B verify -P<profiles>`.
2. **The persistence.** One story when it is not the persistence of the base blueprint: apply the
   `persistence-*` blueprint.
3. **Per workflow module, per `bpmn-*` blueprint.** One story for each `bpmn-*` blueprint the model
   of that module needs, in the order of the process. For example "loan-approval: user task
   `assessRisk` after `bpmn-user-task`". The story copies the delta, adapts the integration test of
   the blueprint to the use case, and adds the attributes of
   [Step 2](../SKILL.md#step-2-parse-the-bpmn) to the model.
4. **Per task with business logic.** One story for each service task or user task whose business
   code is more than a call. The story writes the code in the service class and a test for it.
5. **Per further workflow module.** Steps 3 and 4 again, for each module. With `module-multi` or
   `module-interaction`, one more story for the way the modules work together.
6. **The last story.** "The process runs end to end": run the whole build, start the application,
   walk through the process with the logged URLs, and fix every message of the startup validation,
   as steps 6 and 7 of the [Procedure](../SKILL.md#procedure) say.

Keep each story small. A story changes at most about ten files and adds or adapts one test. If a
story is bigger, split it, for example one story per user task instead of one for all user tasks.

A check must fail before the story is done and pass after it. A build that passes from the start
proves nothing. So a check first tests that the new test file exists, then runs the build of the
module:

```bash
test -f loan-approval/src/test/java/blueprint/workflowmodule/loanapproval/AssessRiskIT.java && mvn -B -pl loan-approval -am verify -Pcamunda7
```

Write `plan/INDEX.md`, one story file per line and an empty `plan/QUESTIONS.md`. Use the
[Templates](#templates). Then commit the plan:

```bash
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
2. Read what the update needs:
   - **A newer VanillaBP 2 version.** The release notes of each VanillaBP repository the project
     uses, from the version it has to the one it gets, and the wiki pages they link.
   - **VanillaBP 1 to 2.** The skill `vanillabp-upgrade-v1-to-v2`. Its survey in step 0 is the
     reading. Its list of findings becomes the stories.
   - **A new BPMN scenario or a new workflow module.** Steps 1 to 4 of the
     [Procedure](../SKILL.md#procedure), as for a new application.
3. Cut the work into stories, in this order:
   1. One story that raises the versions in the `pom.xml` files and nothing else. Check:
      `mvn -B test-compile`. It may fail, then the next stories fix it.
   2. One story per workflow module that adapts its code until it compiles and its tests pass.
      Check: `mvn -B -pl <module> -am verify -P<profiles>`.
   3. One story per configuration change, for example a moved key. Check: the build of the module,
      plus a `grep` that the old key is gone, for example
      `! grep -rn 'vanillabp.default-adapter' --include=*.yaml .`.
   4. One story per new BPMN scenario, as for a new application.
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
   git commit -m "S04 loan-approval: user task assessRisk"
   ```

9. Take the next story: go back to step 6 of
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

| Id  | Title                                            | Status      | Check command                                                                                                     |
|-----|--------------------------------------------------|-------------|-------------------------------------------------------------------------------------------------------------------|
| S01 | Base from module-single, placeholders done       | done        | `mvn -B verify -Pcamunda7`                                                                                        |
| S02 | loan-approval: service task retrieveCreditRating | in-progress | `test -f loan-approval/src/test/java/.../CreditRatingTest.java && mvn -B -pl loan-approval -am verify -Pcamunda7` |
| S03 | loan-approval: user task assessRisk              | open        | `test -f loan-approval/src/test/java/.../AssessRiskIT.java && mvn -B -pl loan-approval -am verify -Pcamunda7`     |
| S04 | loan-approval: timer after bpmn-timer            | open        | `sh plan/checks/S04.sh`                                                                                           |
| S05 | The process runs end to end                      | open        | `mvn -B verify -Pcamunda7`                                                                                        |
```

Write the full path in place of `...`. For an update, the head also holds what step 1 of
[Planning an update](#planning-an-update) found: the VanillaBP version before and after, the
adapters and the workflow modules.

### A story file, plan/S03.md

```markdown
# S03 loan-approval: user task assessRisk

Workflow module: loan-approval
Blueprint: bpmn-user-task
Depends on: S02

## Goal

The process stops at the user task "Assess risk". The endpoint of the task completes it, and the
process goes on along the flow the answer chooses.

## Read first

- AGENTS.md of the blueprint bpmn-user-task, section "Adding this blueprint to an existing project"
- vanillabp skill, references/rules.md, section "User tasks get a package of their own"

## Files

- loan-approval/src/main/resources/loan-approval/processes/camunda7/loan_approval.bpmn (change)
- loan-approval/src/main/java/blueprint/workflowmodule/loanapproval/assessrisk/ (new package)
- loan-approval/src/test/java/blueprint/workflowmodule/loanapproval/AssessRiskIT.java (new)

## Steps

- [ ] 1. Copy the integration test of bpmn-user-task to AssessRiskIT.java and rename it to the use
      case. Run the check. It fails.
- [ ] 2. Add the form key of the user task to the BPMN model.
- [ ] 3. Copy the classes of the user task from the blueprint into the package assessrisk.
- [ ] 4. Replace the placeholders in the copied classes.
- [ ] 5. Run the check. It passes.

## Notes

(Written while working: what is done, what was tried, what is left.)
```

Write the steps so concrete that a weaker model can do each of them without thinking about the
whole application. One step is one file or one test, not "implement the process".

### A check script, plan/checks/S04.sh

```sh
#!/bin/sh
set -e
test -f loan-approval/src/test/java/blueprint/workflowmodule/loanapproval/ReminderTimerIT.java
grep -q 'timeDuration' loan-approval/src/main/resources/loan-approval/processes/camunda7/loan_approval.bpmn
mvn -B -pl loan-approval -am verify -Pcamunda7
```

### An entry of plan/QUESTIONS.md

```markdown
## ASK-003 Which flow of the gateway "Risk acceptable?" is the default?

Story: S03
Date: 2026-11-19
Default taken: "no", the flow which rejects the loan request.
Why this default: it spends no money and commits to nothing.
Answer: open
```

`Answer: open` stays until the user answers. Then it holds the answer, for example
`Answer: yes, accept is the default (2026-11-21)`.
