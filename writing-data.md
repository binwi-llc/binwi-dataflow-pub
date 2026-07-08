# Writing Data

- [Introduction](#introduction)
- [Inserts](#inserts)
- [Inserts without a Generated Key](#inserts-without-a-generated-key)
- [Batch Inserts](#batch-inserts)
- [Timestamps](#timestamps)
- [Updates](#updates)
- [Deletes](#deletes)
- [Increment and Decrement](#increment-and-decrement)
- [Safety Rules for Mutations](#safety-rules-for-mutations)

<a name="introduction"></a>
## Introduction

DataFlow's write API mirrors its read API: start from `db.table(...)`, add constraints
where relevant, and finish with a terminal method. Writes carry extra safety rails —
destructive operations refuse to run without a `WHERE` clause — which this guide covers in
detail.

<a name="inserts"></a>
## Inserts

`insert` takes a `Map` of column names to values, runs the insert, and returns the
generated primary key:

```java
Long id = db.table("users").insert(Map.of(
        "name", "Alice",
        "email", "alice@example.com"
));
```

The key column defaults to `id`. If your table's auto-generated key has a different name,
declare it with `withGeneratedKey` before inserting:

```java
Long id = db.table("accounts")
        .withGeneratedKey("account_id")
        .insert(Map.of("name", "Acme"));
```

> [!NOTE]
> `withGeneratedKey` is reset to `id` on every `table(...)` call, so set it fresh on each
> chain that needs it.

If the driver does not return a usable generated key, `insert` throws an
`InvalidStateException`.

<a name="inserts-without-a-generated-key"></a>
## Inserts without a Generated Key

For tables with composite or client-supplied primary keys — or when you simply don't need
the key back — use `insertWithoutId`. It skips the generated-key lookup and returns the
number of affected rows:

```java
int rows = db.table("session_audit").insertWithoutId(Map.of(
        "session_id", sessionId,
        "ip", ip,
        "ts", Instant.now()
));
```

<a name="batch-inserts"></a>
## Batch Inserts

Insert many rows in one round trip with `insertBatch`. It returns an `int[]` of per-row
affected counts:

```java
db.table("posts").insertBatch(List.of(
        Map.of("user_id", 1L, "title", "Hello"),
        Map.of("user_id", 1L, "title", "World")
));
```

> [!NOTE]
> The column set is taken from the first row, so every row in the batch should share the
> same columns.

<a name="timestamps"></a>
## Timestamps

Call `withCreatedAt()` and/or `withUpdatedAt()` to have DataFlow populate `created_at` and
`updated_at` automatically with the current timestamp:

```java
db.table("users")
    .withCreatedAt()
    .withUpdatedAt()
    .insert(Map.of("name", "Alice"));
```

These flags apply to `insert`, `insertWithoutId`, and `insertBatch`. On `update`,
`withUpdatedAt()` refreshes `updated_at`. If you supply the column explicitly in your map,
your value wins — DataFlow only fills in what's absent.

<a name="updates"></a>
## Updates

`update` takes a `Map` of columns to new values and returns the number of rows changed. It
**requires** a `WHERE` clause:

```java
int updated = db.table("users")
        .where("id", 42L)
        .withUpdatedAt()
        .update(Map.of("name", "Alice 2"));
```

To keep updates predictable and safe, `update` refuses several things:

- No `WHERE` clause → `UnsafeOperationException`.
- A join, `GROUP BY`, `ORDER BY`, `LIMIT`, or `OFFSET` present → `UnsafeOperationException`.
- Any attempt to modify the `id` column → `UnsafeOperationException`.

<a name="deletes"></a>
## Deletes

`delete` removes matching rows and returns the count. Like `update`, it requires a `WHERE`
clause and refuses joins, grouping, ordering, and pagination:

```java
db.table("sessions").where("expires_at", "<", now).delete();
```

<a name="increment-and-decrement"></a>
## Increment and Decrement

Adjust a numeric column in place with `increment` and `decrement`. Both require a `WHERE`
clause:

```java
db.table("users").where("id", 42L).increment("score", 10L);
db.table("users").where("id", 42L).decrement("lives", 1L);
```

The amount is a `Long` bound as a parameter. If `withUpdatedAt()` is set, `updated_at` is
refreshed at the same time.

<a name="safety-rules-for-mutations"></a>
## Safety Rules for Mutations

DataFlow treats destructive operations conservatively. The rules are worth internalizing:

| Operation | Refuses when |
|-----------|--------------|
| `update` | no `WHERE`; join/group/order/limit/offset present; modifying `id` |
| `delete` | no `WHERE`; join/group/order/limit/offset present |
| `increment` / `decrement` | no `WHERE`; join/group/order/limit/offset present |

Each refusal throws an `UnsafeOperationException` — a subclass of `AppException`. These
checks exist to make an accidental full-table `UPDATE` or `DELETE` impossible through the
fluent API. If you genuinely need to bypass them, `unsafeQuery` is the deliberate escape
hatch — see [Raw SQL Escape Hatches](raw-sql-escape-hatches.md) and
[Safety & Exceptions](safety-and-exceptions.md).
</content>
