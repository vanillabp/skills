# Data access

Nothing in MongoDB changes. Same database, same collections, same documents, same indexes. Only
the driver and the Spring Data types on top of it change, so the upgrade needs no data migration
and can be rolled back by rolling back the application.

## Repositories

| Before | After |
|---|---|
| `ReactiveMongoRepository<T, ID>` | `MongoRepository<T, ID>` |
| `ReactiveCrudRepository<T, ID>` | `ListCrudRepository<T, ID>` |
| `ReactiveSortingRepository<T, ID>` | `ListPagingAndSortingRepository<T, ID>` |

Query methods keep their names and their derivation rules. Their return types collapse:
`Mono<T>` becomes `Optional<T>` or `T`, `Flux<T>` becomes `List<T>`, and a `Mono<Void>` delete
becomes `void`. A method returning `Mono<Long>` for a count returns `long`. Methods carrying an
`@Aggregation` annotation keep the annotation verbatim and only change their return type.

The five repositories of the container went that way, and each one changed on two lines, the
`extends` clause and the return type:

```java
// before
@Repository
public interface UserTaskRepository extends ReactiveMongoRepository<UserTask, String> {

    @Aggregation({ ... })
    Flux<UserTask> findAllWorkflowModulesAndUris();

}

// after
@Repository
public interface UserTaskRepository extends MongoRepository<UserTask, String> {

    @Aggregation({ ... })
    List<UserTask> findAllWorkflowModulesAndUris();

}
```

`findById` is the inherited method worth looking at twice. It returns `Optional<T>` where the
reactive one returned `Mono<T>`, so a service wrapping it now has to decide what an absent document
means. The container's services return `null` there, and their callers check for `null` where they
used to chain `switchIfEmpty`.

Custom repository fragments implemented by hand lose their reactive template together with the
interface they implement.

## Templates

`ReactiveMongoTemplate` becomes `MongoTemplate`. The operations have the same names, so a call
loses its terminal operator and nothing else:

```java
// before
final var task = mongoTemplate.findById(id, UserTask.class).block();

// after
final var task = mongoTemplate.findById(id, UserTask.class);
```

`.block()`, `.blockFirst()`, `.blockLast()` and `.toStream()` disappear at the same time as the
publisher they were unwrapping. Where a template call was wrapped in
`Schedulers.boundedElastic()` to keep it off the event loop, that wrapping goes too, because
there is no event loop to protect.

The container declares the template as a bean in `MongoDbConfiguration` and sets the write concern
on it, because optimistic locking needs write result checking. The bean method is called
`mongoTemplate` instead of `reactiveMongoTemplate`, takes a `MongoDatabaseFactory` in place of a
`ReactiveMongoDatabaseFactory`, and keeps the same `WriteResultChecking.EXCEPTION` and
`WriteConcern.MAJORITY` settings. A derived application declaring a template of its own has to keep
doing that.

`OptimisticLockingUtils.doWithRetries` in `commons` was always written against `Supplier`, so it
is unaffected and its call sites stay as they are.

## Changesets

The changeset mechanism itself is unchanged. `@ChangesetConfiguration` on the class,
`@Changeset(order = n)` on the method, the method returning the rollback script or scripts as
`String` or `Collection<String>`, and the applied changesets recorded in their own collection under
an id built from class and method name.

What changes is the parameter the mechanism passes in. `ChangesetAutoConfiguration` reflects over
each `@Changeset` method and hands it a `MongoTemplate` where it used to hand in a
`ReactiveMongoTemplate`. A method taking no parameter at all is still called with none.

```java
// before
@Changeset(order = 1)
public String createMyCollection(
        final ReactiveMongoTemplate mongo) {

    mongo.createCollection(MyEntity.COLLECTION_NAME).block();
    mongo.indexOps(MyEntity.COLLECTION_NAME)
            .ensureIndex(new Index().on("createdAt", Direction.DESC).named(INDEX_CREATED_AT))
            .block();
    return "db." + MyEntity.COLLECTION_NAME + ".drop();";

}

// after
@Changeset(order = 1)
public String createMyCollection(
        final MongoTemplate mongo) {

    mongo.createCollection(MyEntity.COLLECTION_NAME);
    mongo.indexOps(MyEntity.COLLECTION_NAME)
            .ensureIndex(new Index().on("createdAt", Direction.DESC).named(INDEX_CREATED_AT));
    return "db." + MyEntity.COLLECTION_NAME + ".drop();";

}
```

Two rules for this part:

Do not renumber, rename or move a changeset method. Its id is the class name plus the method
name, so a rename makes an applied changeset unknown and a new one appear, and on a database that
already ran it that means creating a collection which exists.

Do not add a new changeset to fix the reactive one. The rewritten method will never run again on
any database that has seen it, which is exactly why rewriting its body is harmless. Add a new
changeset only for a change you actually want applied to existing databases.

## Change streams

`ReactiveChangeStreamUtils` becomes `ChangeStreamUtils`, and the shape of a subscription changes
with it. The reactive version returned a `Flux<ChangeStreamEvent<T>>` which the caller subscribed
to. The blocking one registers a `MessageListener` with a `MessageListenerContainer` and returns a
`Subscription` to cancel with:

```java
// before
changeStreamUtils
        .subscribe(UserTask.class, OperationType.CREATE, OperationType.UPDATE)
        .subscribe(event -> handle(event.getBody()));

// after
final var subscription = changeStreamUtils.subscribe(
        UserTask.class,
        message -> handle(message.getBody()),
        OperationType.CREATE, OperationType.UPDATE);
```

The listener parameter sits before the operation types, and there is an `unsubscribe(Subscription)`
to release a registration. The requirements on the database are unchanged: a replica set, because
that is what change streams need, and the entity class still has to carry the
`public static final String COLLECTION_NAME` field the utility reads by reflection.

The listener runs on a thread of the container's own pool, not on a request thread, so it starts
without a security context. See [behavior-changes.md](behavior-changes.md).

## Lifecycle callbacks

| Before | After |
|---|---|
| `ReactiveBeforeConvertCallback<T>` | `BeforeConvertCallback<T>` |
| `ReactiveAfterConvertCallback<T>` | `AfterConvertCallback<T>` |
| `ReactiveAuditorAware<T>` | `AuditorAware<T>` |

The callback returns the entity instead of a `Publisher` of it:

```java
// before
public Publisher<Object> onBeforeConvert(final Object entity, final String collection) { ... }

// after
public Object onBeforeConvert(final Object entity, final String collection) { ... }
```

`UpdateInformationEventListener` in `commons` does this for `UpdateInformationAware` entities and
stamps the current user into every write. It is a `BeforeConvertCallback<Object>` returning the
entity, and it reads the user from `UserContext`, which means it depends on the security context of
the calling thread. A write performed outside a request is stamped with
`UpdateInformationAware.SYSTEM_USER`, as before, because there is no user to find: the listener
catches `BcUnauthorizedException` and falls back to the system user, and it does the same for an
anonymous caller.
