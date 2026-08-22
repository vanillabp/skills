# Where VanillaBP is documented

Blueprints do not repeat reference documentation, they link it, and so should anything you
generate.

| Topic | Where |
|---|---|
| Using the SPI: annotations, aggregates, multi-instance, call activities | [spi-for-java](https://github.com/vanillabp/spi-for-java) |
| Concepts, workflow modules, platform integration, configuration | [adapter-platform-integration wiki](https://github.com/vanillabp/adapter-platform-integration/wiki) |
| Everything specific to one BPMS | the wiki of the respective [BPMS adapter](https://github.com/vanillabp/adapter-platform-integration/wiki/BPMS-adapters) |
| The blueprints themselves | [the organisation page](https://github.com/vanillabp-blueprints) |
| VanillaBP in general | [vanillabp.io](https://www.vanillabp.io) |

## Pages of the platform wiki

| Page | What it answers |
|---|---|
| [Workflow modules](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-modules) | what a module is, where BPMN files are looked for, how name clashes are avoided |
| [Workflow modules in Spring Boot](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-modules-in-Spring-Boot) | declaring and building a module on Spring Boot |
| [Workflow modules in Quarkus](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-modules-in-Quarkus) | the same on Quarkus |
| [Workflow aggregates](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-aggregates) | persistence, what is synchronized to the BPMS, telling the BPMS the aggregate changed |
| [Workflow tasks](https://github.com/vanillabp/adapter-platform-integration/wiki/Workflow-tasks) | the `@WorkflowTask` contract, errors, retries, repeated delivery, versions |
| [Starting workflows](https://github.com/vanillabp/adapter-platform-integration/wiki/Starting-workflows) | starting from the application, workflows the BPMS starts, learning that one ended |
| [Message correlation](https://github.com/vanillabp/adapter-platform-integration/wiki/Message-correlation) | messages, and why a signal is not a message |
| [Viewing workflows](https://github.com/vanillabp/adapter-platform-integration/wiki/Viewing-workflows) | diagram and execution history through one API |
| [Spring Boot integration](https://github.com/vanillabp/adapter-platform-integration/wiki/Spring-Boot-integration) | what the platform provides, and what the outbox guarantees |
| [Quarkus integration](https://github.com/vanillabp/adapter-platform-integration/wiki/Quarkus-integration) | the same for Quarkus, including the build-time checks |
| [BPMS adapters](https://github.com/vanillabp/adapter-platform-integration/wiki/BPMS-adapters) | which adapters exist and where each is documented |
| [BPMS migration](https://github.com/vanillabp/adapter-platform-integration/wiki/BPMS-migration) | running several BPMS side by side and moving workflows between them |
| [Migrating from version 1](https://github.com/vanillabp/adapter-platform-integration/wiki/Migrating-from-version-1) | upgrading a VanillaBP 1 application, covered by the skill `vanillabp-upgrade-v1-to-v2` |

## Adapter wikis

Everything BPMS specific lives here: configuration keys, transactional behavior and the
deviations each engine forces.

- [Camunda 7](https://github.com/camunda-community-hub/vanillabp-camunda7-adapter/wiki)
- [Camunda 8](https://github.com/camunda-community-hub/vanillabp-camunda8-adapter/wiki)
- [bpm-crafters Process-Engine-API](https://github.com/vanillabp/process-engine-api-adapter/wiki)

Each of them has a `Configuration` page and a `Deviations` page. The deviations page is the one
to read when an engine does not behave the way the platform documentation describes.
