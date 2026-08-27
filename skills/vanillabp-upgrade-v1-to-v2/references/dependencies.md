# Dependencies

## Runtime

| | Version 1 | Version 2 |
|---|---|---|
| Java | 17 | 21 |
| Spring Boot | 3.x | 4.1 |
| Quarkus | 3.x | 3.37 |

Spring Boot 4 modularized `spring-boot-autoconfigure`, so some Spring imports of your own code
move. `@EntityScan` for instance goes from `org.springframework.boot.autoconfigure.domain` to
`org.springframework.boot.persistence.autoconfigure`, and the test slices need explicit
dependencies. That is Spring's change and not VanillaBP's, so follow Spring Boot's own
migration guide for it.

## Renamed artifacts

The adapter artifacts were renamed from `<bpms>-spring-boot-adapter` to
`<bpms>-adapter-spring-boot`, and every adapter now has a Quarkus artifact as well.

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
  <version>2.0</version>
</dependency>
```

| BPMS | Spring Boot | Quarkus |
|---|---|---|
| Camunda 7 | `org.camunda.community.vanillabp:camunda7-adapter-spring-boot` | `org.camunda.community.vanillabp:camunda7-adapter-quarkus` |
| Camunda 8 | `org.camunda.community.vanillabp:camunda8-adapter-spring-boot` | `org.camunda.community.vanillabp:camunda8-adapter-quarkus` |
| Process-Engine-API | `io.vanillabp:process-engine-api-adapter-spring-boot` | `io.vanillabp:process-engine-api-adapter-quarkus` |

Adapters are released independently of the platform, so they carry their own version. Let
`io.vanillabp:vanillabp-bom` manage the platform artifacts and pin the adapter version
separately.

## One dependency or two

On Spring Boot the adapter pulls the platform integration in transitively, so one dependency is
enough. On Quarkus both VanillaBP and the adapter are Quarkus extensions, so both are explicit
dependencies.

Application modules that must not depend on a specific BPMS use the support modules
`io.vanillabp:vanillabp-spring-boot-support` respectively `io.vanillabp:vanillabp-quarkus-support`,
which bring the business SPI `io.vanillabp:vanillabp-integration-spi` transitively.

## Persistence needs no dependency change

Version 1 persisted workflow aggregates through Spring Data, either JPA or MongoDB, and a
repository for the aggregate had to exist. Version 2 does both out of the box, so an upgraded
project changes nothing here. On Quarkus the built-in support is wider than it was: an aggregate
managed by a Panache repository, written as a Panache active record for JPA or MongoDB, or
managed by a Spring Data repository is persisted by VanillaBP itself.

A persistence of your own is new in version 2. Its SPI,
`io.vanillabp.integration.spi.AggregatePersistenceAware`, did not exist in version 1, so no
version 1 project implements it and the upgrade never has to move or rename it. It becomes
interesting only if you want custom persistence now that you can have it.
