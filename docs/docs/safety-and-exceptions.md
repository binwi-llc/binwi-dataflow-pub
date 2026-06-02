# Safety & Exceptions

DataFlow is designed to make SQL injection through its public API structurally difficult. This document explains the guarantees and how to handle failures.

## Safety guarantees

### 1. Identifier validation

Every table name, column name, alias, and join identifier passed to the fluent API is validated **before** SQL is built.

| Dialect | Pattern |
|---------|---------|
| Generic | `^[a-z][a-z0-9_]*$` |
| All others | `^[A-Za-z_][A-Za-z0-9_]*$` |

Qualified names allow one dot (`users.id`). Aliases use `column as alias` in SELECT.

**Effect:** User input cannot become a table or column name unless it passes validation.

### 2. Parameterized values

All dynamic values — including LIMIT and OFFSET — flow through Spring's `NamedParameterJdbcTemplate`:

```sql
WHERE active = :active_1
LIMIT :_limit OFFSET :_offset
```

Values are never concatenated into SQL strings by the fluent API.

### 3. Destructive operation guards

| Operation | Guard |
|-----------|-------|
| `update()` | Requires WHERE; no joins/GROUP BY/ORDER BY/LIMIT; cannot update `id` |
| `delete()` | Requires WHERE; no joins/GROUP BY/ORDER BY/LIMIT |
| `increment()` / `decrement()` | Requires WHERE; no joins/GROUP BY/ORDER BY/LIMIT |
| SQL Server pagination | Requires ORDER BY when using `take()`/`skip()` |

Violations throw `UnsafeOperationException`.

### 4. LIKE wildcard guard

`like()` and `orLike()` reject `%` and `_` in the user-provided search term:

```java
// InvalidArgumentException
db.table("posts").like("title", "50% off", LikeMode.BOTH);
```

Wildcards are added only by `LikeMode`, not by user input.

### 5. Empty IN list rejection

```java
// InvalidArgumentException — never silently ignored
db.table("users").whereIn("id");
```

### 6. Strict groupBy / orderBy

Invalid identifiers throw `InvalidIdentifierException` instead of being silently skipped. Empty `groupBy()` throws `InvalidArgumentException`.

### 7. Raw expression hardening

Raw helpers reject `;`, `--`, `/*`, `*/` and validate bind maps. See [Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

### 8. unsafeQuery is explicit

The only method that skips library validation is `unsafeQuery`. The name documents the risk.

## Exception hierarchy

All DataFlow exceptions extend `com.binwi.dataflow.exception.AppException`:

```
AppException
├── InvalidIdentifierException
├── InvalidArgumentException
├── InvalidBindException
├── InvalidStateException
└── UnsafeOperationException
```

Catch `AppException` to handle any DataFlow failure, or catch specific subclasses for targeted recovery.

## Exception reference

### InvalidIdentifierException

Thrown when a table, column, alias, operator, or joiner fails validation.

```java
try {
    db.table("Users").get();  // GenericDialect — uppercase rejected
} catch (InvalidIdentifierException e) {
    // switch to PostgresDialect or rename table
}
```

Common triggers:

- Invalid table/column name for active dialect
- Unsupported WHERE operator
- Using `where("col", "in", ...)` instead of `whereIn`
- Invalid join table or column

### InvalidArgumentException

Thrown for null/empty inputs, invalid pagination, LIKE wildcard in user value, forbidden raw tokens, empty batch/insert map.

```java
try {
    db.table("users").take(-1).get();
} catch (InvalidArgumentException e) {
    // limit must be >= 0
}
```

### InvalidBindException

Thrown when raw expression binds are missing, unused, malformed, or conflict with prior values.

```java
try {
    db.table("users")
        .whereRaw("x = :a", Map.of("b", 1))
        .get();
} catch (InvalidBindException e) {
    // bind key mismatch
}
```

### InvalidStateException

Thrown when the builder is in an invalid state:

- `get()` / `insert()` called without `table()`
- Unbalanced `groupStart()` / `groupEnd()`
- Insert did not return a usable generated key

### UnsafeOperationException

Thrown when a destructive or unsupported operation is refused:

```java
try {
    db.table("users").update(Map.of("active", false));
} catch (UnsafeOperationException e) {
    // no WHERE clause
}
```

## Handling exceptions in application code

### Service layer pattern

```java
@Service
public class UserService {

    private final DB db;

    public User updateUser(Long id, UpdateUserRequest req) {
        try {
            int rows = db.table("users")
                    .where("id", id)
                    .update(Map.of("name", req.name(), "email", req.email()));
            if (rows == 0) {
                throw new UserNotFoundException(id);
            }
            return db.table("users").where("id", id).first(User.class)
                    .orElseThrow(() -> new UserNotFoundException(id));
        } catch (InvalidIdentifierException e) {
            throw new ProgrammingErrorException("Invalid column in update", e);
        } catch (AppException e) {
            throw new DataAccessException("Database query failed", e);
        }
    }
}
```

### API layer mapping

| Exception | Suggested HTTP status |
|-----------|----------------------|
| `InvalidArgumentException` | 400 Bad Request |
| `InvalidIdentifierException` | 500 (programming error) |
| `InvalidBindException` | 500 (programming error) |
| `UnsafeOperationException` | 500 (programming error) |
| `InvalidStateException` | 500 (programming error) |

User-facing validation should happen before calling DataFlow; library exceptions usually indicate bugs or misconfiguration.

## What DataFlow does not protect against

1. **unsafeQuery** with attacker-controlled SQL structure
2. **Raw expressions** with logically wrong but syntactically valid SQL
3. **Denial of service** — no query timeout or row limit enforcement beyond what you set
4. **Authorization** — no row-level security; enforce in application layer
5. **Race conditions** — use transactions and appropriate isolation levels

## Security checklist

- [ ] Use fluent API or raw helpers — avoid `unsafeQuery` for user-influenced SQL
- [ ] Keep user input in bind values, not in identifiers or raw SQL structure
- [ ] Require WHERE on all UPDATE/DELETE paths in code review
- [ ] Use appropriate dialect for your schema
- [ ] Close streams promptly to avoid connection exhaustion
- [ ] Do not share mutable `DB` instances across threads
