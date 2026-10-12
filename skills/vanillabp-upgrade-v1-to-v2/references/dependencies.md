# Dependencies

## Runtime

| | Version 1 | Version 2 |
|---|---|---|
| Java | 17 | 21 |
| Spring Boot | 3.x | 4.1 |
| Quarkus | 3.x | 3.37 |

Spring Boot 4 split `spring-boot-autoconfigure` into modules. So some Spring imports of your
own code move. `@EntityScan`, for example, goes from
`org.springframework.boot.autoconfigure.domain` to
`org.springframework.boot.persistence.autoconfigure`. The test slices need explicit
dependencies. That is Spring's change, not VanillaBP's. Follow Spring Boot's own migration
guide for it.

## Renamed artifacts

The adapter artifacts were renamed from `<bpms>-spring-boot-adapter` to
`<bpms>-adapter-spring-boot`. Every adapter now has a Quarkus artifact as well.

```xml
<!-- before (Spring Boot, Camunda 7) -->
<dependency>
  <groupId>org.camunda.community.vanillabp</groupId>
  <artifactId>camunda7-spring-boot-adapter</artifactId>
  <version>1.x.x</version>
</dependency>

<!-- after -->
<dependency>
  <groupId>org.camunda.community.vanillabp</groupId>
  <artifactId>camunda7-adapter-spring-boot</artifactId>
  <version>2.0.0</version>
</dependency>
```

| BPMS | Spring Boot | Quarkus | Version |
|---|---|---|---|
| Camunda 7 | `org.camunda.community.vanillabp:camunda7-adapter-spring-boot` | `org.camunda.community.vanillabp:camunda7-adapter-quarkus` | `2.0.0` |
| Camunda 8 | `org.camunda.community.vanillabp:camunda8-adapter-spring-boot` | `org.camunda.community.vanillabp:camunda8-adapter-quarkus` | `2.0.0` plus the cluster line, e.g. `2.0.0-8.9` |
| Process-Engine-API | `io.vanillabp:process-engine-api-adapter-spring-boot` | `io.vanillabp:process-engine-api-adapter-quarkus` | `2.0.0` |

Adapters are released on their own, apart from the platform, so they carry their own version.
Let `io.vanillabp:vanillabp-bom` manage the platform artifacts, and pin the adapter version
separately.

## The Camunda 8 version carries the cluster line

The Camunda 8 adapter is published once per minor version of Camunda 8, and that minor version
is part of its version. A coordinate written without it resolves to nothing:

```xml
<dependency>
  <groupId>org.camunda.community.vanillabp</groupId>
  <artifactId>camunda8-adapter-spring-boot</artifactId>
  <version>2.0.0-8.9</version>
</dependency>
```

The cluster the application talks to decides which suffix belongs there. Camunda promises that
a client works against clusters of its own version and newer. It says nothing about the other
direction. So the client a build was compiled against is the lowest cluster version that build
accepts. The rule which follows: take the line whose client version is at or below your
cluster's minor version.

Today there are three lines. `-8.10` is the current one and gets everything, `-8.8` and `-8.9`
get bugfixes only. A preview line, built against a pre-release of the next minor version, joins
them while such a pre-release exists. There is none at the moment. So a cluster on 8.10 or newer
asks for `-8.10`, one on 8.9 for `-8.9`, and one on 8.8 for `-8.8`. Read the lines and their
client versions from the table in the
[adapter's wiki](https://github.com/camunda-community-hub/vanillabp-camunda8-adapter/wiki#which-release-line-to-use),
not from here. A line lives as long as Camunda maintains its minor version, not only until the
next minor version goes GA.

Nothing in your code depends on which line you pick. The methods are the same on all of them,
and a call your cluster cannot serve fails with a message which names your line. Moving to
another line means upgrading the cluster. So it is one decision, not two.

Where the project uses Renovate, extend the preset the adapter ships
(`github>camunda-community-hub/vanillabp-camunda8-adapter//renovate/camunda8-lines.json`). It
reads the suffix as a compatibility value and never changes it on its own. So no automatic
update moves the application to a minor version of the cluster it does not run.

## One dependency or two

On Spring Boot the adapter brings the platform integration along, so one dependency is enough.
On Quarkus both VanillaBP and the adapter are Quarkus extensions, so both are explicit
dependencies.

Application modules which must not depend on a specific BPMS use the support modules
`io.vanillabp:vanillabp-spring-boot-support` or `io.vanillabp:vanillabp-quarkus-support`. They
bring the business SPI `io.vanillabp:vanillabp-integration-spi` along.

## Persistence needs no dependency change

Version 1 stored workflow aggregates through Spring Data, either JPA or MongoDB, and a
repository for the aggregate had to exist. Version 2 does both out of the box, so an upgraded
project changes nothing here. On Quarkus the built-in support is wider than it was. VanillaBP
itself stores an aggregate which a Panache repository manages, one written as a Panache active
record for JPA or MongoDB, and one which a Spring Data repository manages.

A persistence of your own is new in version 2. Its SPI,
`io.vanillabp.integration.spi.AggregatePersistenceAware`, did not exist in version 1. So no
version 1 project implements it, and the upgrade never has to move or rename it. It only
becomes interesting if you want a persistence of your own, now that you can have it.
