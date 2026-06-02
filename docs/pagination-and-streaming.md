# Pagination & Streaming

## Pagination with `paginate()`

`paginate(perPage, page)` executes two queries: a `COUNT(*)` for the total, then a limited SELECT for the page. Page numbers are **1-based**.

```java
Page<Map<String, Object>> page = db.table("users")
    .where("active", true)
    .orderByDesc("created_at")
    .paginate(20, 1);
```

### Page record fields

`Page<T>` is an immutable record in `com.binwi.dataflow.Page`:

| Method | Type | Description |
|--------|------|-------------|
| `items()` | `List<T>` | Rows for this page (unmodifiable copy) |
| `total()` | `long` | Total matching rows (ignores LIMIT/OFFSET) |
| `page()` | `int` | Current page number (1-based) |
| `perPage()` | `int` | Requested page size |
| `totalPages()` | `int` | `(total + perPage - 1) / perPage` |
| `hasNext()` | `boolean` | More pages after this one |
| `hasPrevious()` | `boolean` | `page > 1` |

### Iterating a page

`Page<T>` implements `Iterable<T>`:

```java
for (Map<String, Object> row : page) {
    process(row);
}
```

### Typed pagination

```java
Page<User> page = db.table("users")
    .orderByDesc("created_at")
    .paginate(20, 1, User.class);

Page<String> emails = db.table("users")
    .select("email")
    .paginate(50, 2, (rs, n) -> rs.getString("email"));
```

### Validation

- `perPage` and `page` must both be `> 0`
- Violations throw `InvalidArgumentException`

### REST API example

```java
@GetMapping("/users")
public PageResponse listUsers(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "20") int perPage) {

    Page<User> result = db.table("users")
            .where("active", true)
            .orderByAsc("name")
            .paginate(perPage, page, User.class);

    return new PageResponse(
            result.items(),
            result.total(),
            result.page(),
            result.perPage(),
            result.totalPages(),
            result.hasNext(),
            result.hasPrevious()
    );
}
```

### Pagination and dialects

| Dialect | SQL emitted |
|---------|-------------|
| Generic, Postgres, MySQL, MariaDB, H2 | `LIMIT :_limit OFFSET :_offset` |
| Oracle, SQL Server | `OFFSET :_offset ROWS FETCH NEXT :_limit ROWS ONLY` |

On SQL Server, ensure an `ORDER BY` is present — otherwise `UnsafeOperationException` is thrown when pagination is applied.

## Manual LIMIT/OFFSET with `take()` and `skip()`

For simpler cases without a total count:

```java
List<Map<String, Object>> rows = db.table("users")
    .orderByAsc("id")
    .take(20)
    .skip(40)
    .get();
```

| Method | Bind parameter | SQL concept |
|--------|----------------|-------------|
| `take(n)` | `:_limit` | LIMIT / FETCH NEXT |
| `skip(n)` | `:_offset` | OFFSET |

Both reject negative values.

`first()` internally sets `take(1)`.

## Streaming large result sets

For result sets too large to fit in memory, use `stream()`:

```java
try (Stream<User> users = db.table("users")
        .where("active", true)
        .orderByAsc("id")
        .stream(User.class)) {
    users.forEach(this::processUser);
}
```

### Overloads

| Method | Returns |
|--------|---------|
| `stream()` | `Stream<Map<String, Object>>` |
| `stream(Class<T>)` | `Stream<T>` via `DataClassRowMapper` |
| `stream(RowMapper<T>)` | `Stream<T>` via custom mapper |

### Critical: close the stream

Streams are backed by a live JDBC cursor. **Always** close them:

```java
// Preferred — try-with-resources
try (Stream<User> s = db.table("users").stream(User.class)) {
    s.forEach(handler);
}

// Or explicit close
Stream<User> s = db.table("users").stream(User.class);
try {
    s.forEach(handler);
} finally {
    s.close();
}
```

Failure to close can leak connections from the pool.

### Streaming vs `get()`

| | `get()` | `stream()` |
|---|---------|------------|
| Memory | Loads all rows | Cursor-based, constant memory |
| Connection held | Until list returned | Until stream closed |
| Use case | Small/medium sets | Exports, ETL, large reports |

### Streaming within a transaction

When streaming inside `@Transactional`, keep the transaction open until the stream is fully consumed and closed:

```java
@Transactional(readOnly = true)
public void exportUsers(Consumer<User> consumer) {
    try (Stream<User> users = db.table("users").stream(User.class)) {
        users.forEach(consumer);
    }
}
```

## Count without pagination

When you only need the total:

```java
long total = db.table("users").where("active", true).count();
```

`count()` does not apply `take()`/`skip()` from a previous builder state on the same chain — it builds a dedicated count query and leaves the builder unchanged for subsequent calls on the same chain.

Note: after `paginate()`, the builder's `take`/`skip` are set as a side effect. Start a new chain with `table()` for unrelated queries.

## Performance notes

1. **`paginate()` runs COUNT + SELECT** — for infinite scroll UIs that do not need exact totals, use `take(perPage + 1)` and infer `hasNext` from row count
2. **Index ORDER BY columns** used with pagination
3. **Deep OFFSET** can be slow on large tables — consider keyset pagination via `where("id", ">", lastId)` for very large datasets
4. **Streams hold connections** — process and close promptly

## Keyset pagination pattern

DataFlow does not have built-in keyset pagination, but you can implement it:

```java
List<User> page = db.table("users")
    .where("active", true)
    .where("id", ">", lastSeenId)
    .orderByAsc("id")
    .take(pageSize)
    .get(User.class);

boolean hasMore = page.size() == pageSize;
```
