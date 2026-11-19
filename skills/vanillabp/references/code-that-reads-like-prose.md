# Code that reads like prose

*Part of the skill [`vanillabp`](../SKILL.md).*

The people who keep a process application running did not write it. They come to one class to fix
one bug, and they do not know the rest. So the code has to explain itself where they land. These
rules hold for everything you write: the Java code, the tests, the BPMN names you choose, the
comments, the log messages, the configuration comments and the `README.md` files.

The blueprints follow the same rules. Where this page is short, read them there:

- [Rule 2: a blueprint shows one aspect](https://github.com/vanillabp-blueprints/blueprints/blob/main/CONTRIBUTING.md#rule-2-a-blueprint-shows-one-aspect),
  for the names of the beans,
- [Writing style](https://github.com/vanillabp-blueprints/blueprints/blob/main/CONTRIBUTING.md#writing-style)
  and [What a blueprint may point at](https://github.com/vanillabp-blueprints/blueprints/blob/main/CONTRIBUTING.md#what-a-blueprint-may-point-at),
- [How we write](https://github.com/vanillabp-blueprints/blueprints/blob/main/AGENTS.md#how-we-write).

**Contents:**

1. [Names read like small sentences](#names-read-like-small-sentences)
1. [A name says what a thing stands for](#a-name-says-what-a-thing-stands-for)
1. [A comment says why](#a-comment-says-why)
1. [Nothing points at the plan](#nothing-points-at-the-plan)
1. [English for second-language readers](#english-for-second-language-readers)
1. [Nothing a program reads is renamed](#nothing-a-program-reads-is-renamed)
1. [The checks before each commit](#the-checks-before-each-commit)

## Names read like small sentences

A method name says what the method does. A call then reads like a short sentence about the business
case, and a method body, with its comments, reads like a short text. `assessCreditRating`,
`loanRequested` and `isRiskAcceptable` are names. `handle`, `process` and `doTask` are not.

A test class says which behaviour it tests, for example `AssessRiskIT` or
`ReminderAfterThreeDaysIT`. A test method says what it proves, for example
`deniedRequestEndsWithoutPayout`.

## A name says what a thing stands for

Name a field, a parameter or a variable after the role it plays in the class, not after its type.
The blueprints use the same names everywhere:

| Bean                     | Name                                    |
|--------------------------|-----------------------------------------|
| the use case's `Service` | the use case, singular: `loanApproval`  |
| `ProcessService`         | `bpms`                                  |
| `Workflow`               | `workflow`                              |
| the aggregate repository | what it stores, plural: `loanApprovals` |

So a call reads like a sentence:

```java
@Autowired
private Service loanApproval;

@WorkflowTask
public void retrieveCreditRating(final Aggregate loanRequest) {
  loanApproval.assessCreditRating(loanRequest);
}
```

```java
@Autowired
private ProcessService<Aggregate> bpms;

public void loanRequested(final Aggregate loanApproval) {
  bpms.startWorkflow(loanApproval);
}
```

Another use case takes its own word, such as `rateWatch` or `nightlyReview`. Where a parameter would
hide the field, give it a word from the domain: the aggregate of a loan approval is `loanRequest`.
Avoid `request` and `application` alone, because both words already mean something else in IT.

If a method name would repeat the field, drop that part: `loanApproval.request(...)` and
`loanApproval.get(id)`, not `loanApproval.initiateLoanApproval(...)` and
`loanApproval.getLoanApproval(id)`.

A type name inside a name is the sign to look for: `loanApprovalService`, `processService`. So are
filler words such as `data`, `result`, `value`, `list`, `obj` and `tmp`. The one exception: two
variables of the same type that stand for different things. Even there, name the roles.

## A comment says why

A comment that repeats what the line says is noise. Leave it out. A comment that explains why the
obvious way is not the one taken is the reason comments exist.

Before you write a comment, try a better name. A name that carries the meaning is better than a
comment next to a bad name.

A comment must be complete where it stands. "Same reason as above" and "see the other module" are
not complete, because code moves and the other place changes.

```java
// not like this
// check the amount
if (loanRequest.getAmount().compareTo(LIMIT) > 0) {

// like this
// The bank lets one person decide on small loans only. Above this amount a second
// person assesses the risk as well. This is a rule of the business, not of the code.
if (loanRequest.getAmount().compareTo(LIMIT_FOR_ONE_ASSESSOR) > 0) {
```

Where VanillaBP behaves in a way the reader needs to know, link the wiki page that describes it.
Never link an issue, a pull request or the decision log of a VanillaBP repository. Where no wiki
page says it, say it in your own words.

## Nothing points at the plan

The code, the tests, the comments and the texts of the application never name a story id such as
`S04`, a question id such as `ASK-003`, a ticket, an issue, a pull request, a chat or a person.
Those record a moment in time. A later change can make them wrong, and nothing in the build notices.
Where the reason matters, write the reason itself.

A commit message may name the story id. A commit is itself a record of a moment in time.

## English for second-language readers

Most people who read the code read English as a second language. Write so that nobody has to read a
sentence twice.

- Short main sentences, one thought each. One subordinate clause is enough.
- Active voice: "The workflow starts the task", not "The task is started".
- The common word instead of the rare one: `use`, not `leverage`; `about`, not `regarding`; `so`,
  not `consequently`; `before`, not `prior to`.
- Not three or more nouns in a row, such as `loan request risk assessment result`.
- Write an abbreviation out the first time, and say what a technical term means the first time it
  turns up.
- No em dashes, no lists whose items start with a bold label, no phrases about how important
  something is.

## Nothing a program reads is renamed

These rules are for the names you choose and the sentences you write. Nothing a program reads is
renamed for the sake of language: the ids in a BPMN model that the code refers to, the BPMN process
id, the workflow module id, configuration keys, REST paths, Maven coordinates, table names and the
names of classes and methods that other code or tests already use. A BPMN id that a business
analyst gave stays as it is, because the model and the code have to match.

## The checks before each commit

Run these on the change of the story, after `git add -A`. They leave out the folder `plan/`,
because the plan names its stories on purpose.

```bash
git diff --cached -- . ':(exclude)plan' | grep -niE '^\+.*\b(story|prompt|issue|ticket|jira)s? ?#?[0-9]+'
git diff --cached -- . ':(exclude)plan' | grep -nE '^\+.*\b(S[0-9]{2}|ASK-[0-9]{3})\b'
git diff --cached -- . ':(exclude)plan' | grep -nE '^\+' | grep -oE '[^.!?]{160,}'
git diff --cached -- . ':(exclude)plan' | grep -niE '^\+.*\b(leverage|utilize|regarding|consequently|possesses|facilitate|prior to|aforementioned|thereby|whilst|in order to)\b'
git diff --cached -- . ':(exclude)plan' | grep -nE '^\+.*\b(private|final) [A-Za-z<>]+ [a-z][A-Za-z]*(Service|Repository|Data|Result|Value|List|Obj|Tmp)\b'
```

Each line that a command prints is a place to read again. Fix the place, or leave it when the rule
does not apply there, for example a quoted error message. The commands do not find everything, so
also read each new name and each new comment once.
