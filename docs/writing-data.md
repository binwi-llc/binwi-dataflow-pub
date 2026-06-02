# Writing Data

DataFlow supports INSERT, UPDATE, DELETE, and atomic increment/decrement operations. All values are bound via named parameters.

## INSERT

### Single row with generated key

Use when the table has an auto-generated primary key and you need the new `id`:

```java
Long id = db.table("users")
    .insert(Map.of(
        "name", "Alice",
        "email", "alice@example.com",
        "active", true
    ));
```

DataFlow expects the generated key column to be named `id`. If the driver returns keys under a different name, retrieval may fail with `InvalidStateException`.

### Single row without generated key

For composite keys, client-supplied IDs, or when you do not need the key back:

```java
int rows = db.table("session_audit")
    .insertWithoutId(Map.of(
        "session_id", sessionId,
        "ip", ip,
        "ts", Instant.now()
    ));
// rows == 1 on success
```

### Automatic timestamps

```java
Long id = db.table("users")
    .withCreatedAt()
    .withUpdatedAt()
    .insert(Map.of("name", "Alice", "email", "alice@example.com"));
```

- `withCreatedAt()` adds `created_at` if absent (current timestamp)
- `withUpdatedAt()` adds `updated_at` if absent (current timestamp)

Both can be combined on insert. Column names must exist in your schema and match identifier rules.

### Batch insert

```java
int[] counts = db.table("posts")
    .withCreatedAt()
    .insertBatch(List.of(
        Map.of("user_id", 1L, "title", "Hello"),
        Map.of("user_id", 1L, "title", "World")
    ));
```

Requirements:

- All rows must share the same column set (derived from the first row)
- Empty list throws `InvalidArgumentException`
- Timestamps from `withCreatedAt()` / `withUpdatedAt()` are applied to every row

## UPDATE

```java
int updated = db.table("users")
    .where("id", 42L)
    .withUpdatedAt()
    .update(Map.of(
        "name", "Alice Updated",
        "email", "alice.new@example.com"
    ));
```

### Safety rules

UPDATE is refused (throws `UnsafeOperationException`) when:

| Condition | Reason |
|-----------|--------|
| No WHERE clause | Prevents mass updates |
| Joins present | Cross-table updates not supported |
| GROUP BY present | Invalid UPDATE shape |
| ORDER BY present | Invalid UPDATE shape |
| LIMIT/OFFSET set | Invalid UPDATE shape |
| `id` in update map | Primary key mutation refused |

Always include at least one `where(...)` before calling `update()`.

### Example: conditional update

```java
int rows = db.table("users")
    .where("id", userId)
    .where("active", true)
    .update(Map.of("last_login", Timestamp.from(Instant.now())));
```

## DELETE

```java
int deleted = db.table("sessions")
    .where("expires_at", "<", Timestamp.from(Instant.now()))
    .delete();
```

### Safety rules

DELETE is refused when:

| Condition | Reason |
|-----------|--------|
| No WHERE clause | Prevents full-table delete |
| Joins, GROUP BY, ORDER BY, LIMIT/OFFSET | Unsupported DELETE shapes |

```java
// Throws UnsafeOperationException
db.table("users").delete();

// Safe
db.table("users").where("id", 42L).delete();
```

## INCREMENT and DECREMENT

Atomically adjust a numeric column:

```java
db.table("users").where("id", 42L).increment("score", 10L);
db.table("users").where("id", 42L).decrement("lives", 1L);
```

Rendered SQL:

```sql
UPDATE users SET score = score + :incval WHERE id = :id_1
```

Rules:

- WHERE clause required
- Value must be non-null
- No joins, GROUP BY, ORDER BY, or pagination
- `withUpdatedAt()` also updates `updated_at` when enabled

## Column validation

Every column name in insert/update maps is validated against the active dialect. Invalid names throw `InvalidIdentifierException` before SQL is sent to the database.

## Transactions

Combine multiple writes in a single transaction:

```java
@Transactional
public void transferPoints(Long fromId, Long toId, long amount) {
    db.table("users").where("id", fromId).decrement("points", amount);
    db.table("users").where("id", toId).increment("points", amount);
}
```

Ensure your `DB` usage aligns with your transaction boundaries. With prototype-scoped beans, both calls on the same injected `DB` instance in one `@Transactional` method share the same builder object but each `table()` resets state — which is the intended pattern.

## Common patterns

### Upsert alternative

DataFlow does not ship MERGE/ON CONFLICT helpers. Options:

1. `first()` then `insert()` or `update()` in application code
2. `unsafeQuery()` with database-specific upsert SQL (last resort)
3. `whereRaw` + custom SQL via `unsafeQuery`

### Soft delete

```java
db.table("users")
    .where("id", userId)
    .withUpdatedAt()
    .update(Map.of("deleted_at", Timestamp.from(Instant.now())));
```

### Bulk update by IDs

```java
db.table("users")
    .whereIn("id", ids)
    .update(Map.of("active", false));
```

## Error reference

| Exception | Typical write-time cause |
|-----------|-------------------------|
| `UnsafeOperationException` | UPDATE/DELETE without WHERE, or forbidden query shape |
| `InvalidArgumentException` | Empty input map, empty batch |
| `InvalidIdentifierException` | Bad column name in map |
| `InvalidStateException` | Insert did not return generated key |

See [Safety & Exceptions](safety-and-exceptions.md) for the full hierarchy.
