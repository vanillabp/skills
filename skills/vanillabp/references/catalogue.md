# The blueprint catalogue

A blueprint is a small application you can run. It shows one aspect of building business
process software with VanillaBP. They all use the same package structure, the same placeholders
and the same test setup. So if you understand one, you understand all of them. To combine two,
you apply two deltas.

Blueprints are developed in one monorepo. They are delivered as one repository per blueprint
and platform, at `https://github.com/vanillabp-blueprints/<blueprint-id>-<platform>`, where
`<platform>` is `springboot` or `quarkus`. Those delivered repositories are read-only mirrors.
So issues and pull requests belong in
[the monorepo](https://github.com/vanillabp-blueprints/blueprints).

## The catalogue is the only list

Which blueprints exist is written in one file, and nowhere else:
[blueprints.yaml](https://github.com/vanillabp-blueprints/.github/blob/main/blueprints.yaml).
Fetch it in its raw form:

```
https://raw.githubusercontent.com/vanillabp-blueprints/.github/main/blueprints.yaml
```

This skill keeps no copy of it on purpose. A new blueprint is in the catalogue the day it is
delivered, and you use it at once. If you cannot fetch the file, do not pick blueprints from
memory. Ask the user to give you the file, or to let you reach the network.

## What an entry tells you

- `id`: the name of the blueprint, also the first part of its repository name.
- `category`: `module`, `persistence`, `bpmn` and so on. Step 3 of the skill picks one module
  structure, one persistence and every `bpmn` scenario the model needs.
- `covers.bpmn`: the BPMN element types the blueprint shows. Compare them with the element
  types of the model you have to implement.
- `covers.spi`: the SPI members the blueprint uses.
- `base`: the blueprint it builds on. Most entries build on `module-single`. Some build on
  another one: `module-process-driver` builds on `module-simulator`. The deepest taken entry of
  such a chain gives the Maven layout of the project, see
  [project-structure.md](project-structure.md#the-maven-layout-comes-from-the-module--blueprints).
- `composes_with`: the blueprints it combines with.
- `choose_one`: the group of alternatives the entry belongs to, for example `module-structure`
  or `aggregate-storage`. Take at most one entry of each group, and exactly one of
  `module-structure`. Taking none means the default of `module-single`; the description of
  `choose_one` in `blueprints.schema.json` says what that is per group. An entry without it is
  an addition, which you take on top when it applies.
- `bpms`: the engines the blueprint is built and tested against.
- `docs`: the documentation it links.
- `platforms.<platform>`: per platform the `status`, the repository and the URL of its
  `AGENTS.md` (`agents_md`). Use only an entry whose status is `available`.

`platforms` holds more than the two Java platforms. A frontend of the Business Cockpit, an entry
whose id ends in `-frontend`, runs in the browser and builds with npm. So its platforms are the UI
frameworks `react`, `angular` and `vue`. For such an entry, `springboot` and `quarkus` are always
`not-applicable`. That is expected and no reason to leave it out. Look at the key of the UI
framework of your job instead. If the job has no user interface of its own, no frontend entry
applies, and that is the reason to write down.

Several entries usually apply to one model, and that is normal. Each is a delta on top of
`module-single`.

A parallel gateway, a non-interrupting boundary event and a multi-instance element each add a
token to the workflow. So the persistence blueprint about parallel branches matters for each of
them too. Read it before you let two branches write one aggregate.

## Choosing the BPMS

A Maven profile decides which engine a blueprint runs on, never a code change. Keep that wiring
when you copy a blueprint. [project-structure.md](project-structure.md) describes it in the
section about configuration. Which engines a blueprint supports is in its `bpms` field.
