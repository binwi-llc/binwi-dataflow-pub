# Typed Results

DataFlow maps JDBC rows to Java types using Spring's `DataClassRowMapper` and custom `RowMapper` implementations.

## Snake_case to camelCase

Database columns in snake_case automatically map to camelCase record components and bean properties:

| Column | Record component |
|--------|------------------|
| `id` | `id` |
| `created_at` | `createdAt` |
| `user_id` | `userId` |

## Java records (recommended)

```java
public record User(
        Long id,
        String name,
        String email,
        Boolean active,
        Integer age,
        String role,
        Timestamp createdAt,
        Timestamp updatedAt
) {}
```

### List

```java
List<User> users = db.table("users")
    .where("active", true)
    .orderByAsc("name")
    .get(User.class);
```

### Optional single row

```java
Optional<User> user = db.table("users")
    .where("id", 42L)
    .first(User.class);

User u = user.orElseThrow(() -> new NoSuchElementException("user not found"));
```

### Paginated

```java
Page<User> page = db.table("users")
    .orderByDesc("created_at")
    .paginate(20, 1, User.class);
```

## POJOs (mutable beans)

Standard JavaBeans with a no-arg constructor and setters work via the same mapper:

```java
public class User {
    private Long id;
    private String name;
    private String email;

    // getters and setters
}
```

Prefer records for immutable data transfer objects.

## Untyped maps

When you omit a type argument, results are `Map<String, Object>` with **snake_case keys** (as JDBC returns them):

```java
List<Map<String, Object>> rows = db.table("users").get();
String name = (String) rows.get(0).get("name");
Timestamp created = (Timestamp) rows.get(0).get("created_at");
```

In 2.0.0, `paginate()` without a type also returns snake_case maps — consistent with `get()` and `first()`.

## Custom RowMapper

For projections that do not match a class shape:

```java
List<String> emails = db.table("users")
    .select("email")
    .where("active", true)
    .get((rs, rowNum) -> rs.getString("email"));
```

With `first`:

```java
Optional<String> email = db.table("users")
    .where("id", 42L)
    .select("email")
    .first((rs, n) -> rs.getString("email"));
```

With pagination:

```java
Page<EmailView> page = db.table("users")
    .select("id", "email")
    .paginate(50, 1, (rs, n) -> new EmailView(rs.getLong("id"), rs.getString("email")));
```

With streaming:

```java
try (Stream<String> emails = db.table("users")
        .select("email")
        .stream((rs, n) -> rs.getString("email"))) {
    emails.forEach(mailService::validate);
}
```

## Partial column mapping

Select only the columns your type needs:

```java
public record UserSummary(Long id, String name) {}

List<UserSummary> summaries = db.table("users")
    .select("id", "name")
    .get(UserSummary.class);
```

Extra columns in the result set are ignored. Missing columns leave components null (or cause mapper errors for primitives).

## Join result mapping

Qualify selected columns and match names in your record:

```java
public record PostWithAuthor(
        Long id,
        String title,
        String authorName
) {}

// Use alias for joined column
db.table("posts")
    .select("posts.id", "posts.title", "users.name as author_name")
    .innerJoin("users", "users.id", "posts.user_id", on -> {})
    .get(PostWithAuthor.class);
```

The alias `author_name` maps to `authorName`.

## Null handling

- Wrapper types (`Long`, `Integer`, `Boolean`) accept SQL NULL
- Primitive `long`, `int`, `boolean` may fail or default depending on Spring's mapper behaviour — prefer wrapper types in records for nullable columns

## Type compatibility

Common JDBC types map as expected:

| SQL type | Java type |
|----------|-----------|
| BIGINT | `Long` |
| VARCHAR | `String` |
| BOOLEAN | `Boolean` |
| TIMESTAMP | `java.sql.Timestamp` or `Instant` (driver dependent) |
| DECIMAL | `BigDecimal` |

Test mapping for your specific driver and column types in integration tests.

## When typed mapping fails

| Issue | Solution |
|-------|----------|
| Column name mismatch | Add SQL alias: `select("display_name as displayName")` |
| Nested objects | Use `RowMapper` or join in application code |
| Enums | Custom `RowMapper` or `@Enum` converter outside DataFlow |
| JSON columns | Map as `String` then parse, or use custom `RowMapper` |

## Comparison of read methods

| Method | Return type | Empty result |
|--------|-------------|--------------|
| `get()` | `List<Map<...>>` | Empty list |
| `get(User.class)` | `List<User>` | Empty list |
| `first()` | `Optional<Map<...>>` | `Optional.empty()` |
| `first(User.class)` | `Optional<User>` | `Optional.empty()` |
| `stream(User.class)` | `Stream<User>` | Empty stream |
| `paginate(20, 1, User.class)` | `Page<User>` | Page with empty items, total may be 0 |
