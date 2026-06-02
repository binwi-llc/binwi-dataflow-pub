# Binwi DataFlow

A fluent, parameter-safe SQL query builder on top of Spring's `NamedParameterJdbcTemplate`. Write expressive SQL in Java without giving up parameterized queries or auditability of the generated SQL.

```java
List<Map<String, Object>> users = db.table("users")
        .select("id", "name", "email")
        .where("active", true)
        .where("age", ">", 18)
        .orderByDesc("created_at")
        .take(10)
        .get();
```

---

## Documentation

Detailed usage guides live in the [`docs/`](docs/) folder:

- [Getting Started](docs/getting-started.md) — first query in five minutes
- [Full documentation index](docs/README.md) — installation, dialects, joins, pagination, safety, API reference, and more

---

## Features

- Fluent builder for `SELECT` / `INSERT` / `UPDATE` / `DELETE` / `INCREMENT` / `DECREMENT`.
- First-class predicates: `where`, `whereIn` / `whereNotIn` (varargs + `Collection`), `whereBetween`, `whereExists` / `whereNotExists`, `whereNull` / `whereNotNull`, `like` (with `LikeMode`), plus `or*` variants of every one.
- `LEFT` / `RIGHT` / `INNER` joins with a dedicated on-condition DSL (`leftJoin(..., on -> on.where(...).colEq(...))`).
- Parenthesized predicate groups via `groupStart()` / `groupEnd()` and `orGroupStart()`.
- `distinct()` on the SELECT.
- Strict identifier validation by default: every table / column name must match `^[a-z][a-z0-9_]*$` — anything else is rejected before reaching the database. Opt into mixed-case quoted identifiers by selecting a dialect (see below).
- Pluggable `Dialect` strategy with built-in support for Postgres, MySQL, MariaDB, H2, SQL Server, and Oracle. Picking a dialect flips on identifier quoting (`"col"` / `` `col` `` / `[col]`), relaxes validation to `[A-Za-z_][A-Za-z0-9_]*`, and switches pagination syntax where the database demands it.
- Every value flows through `NamedParameterJdbcTemplate` placeholders, including `LIMIT` and `OFFSET` (`:_limit`, `:_offset`). No string concatenation of user data into SQL.
- Raw escape hatches (`selectRaw`, `whereRaw`, `havingRaw`, `joinRaw`, `orderByAscRaw`, `orderByDescRaw`) that reject `;`, `--`, `/*`, `*/` and require named-bind validation when binds are provided.
- Safety rails on destructive operations: `update()`, `delete()`, `increment()`, `decrement()` all refuse to run without a `WHERE`.
- Typed result mapping: `get(MyPojo.class)`, `first(MyPojo.class)` and `paginate(perPage, page, MyPojo.class)` map snake_case columns to camelCase record components / bean properties via Spring's `DataClassRowMapper`. `RowMapper<T>` overloads are also available.
- Streaming reads (`stream()` / `stream(Class)` / `stream(RowMapper)`) backed by a JDBC cursor for iterating large result sets without materialising them.
- Built-in pagination (`paginate(perPage, page)`) returning a typed `Page<T>` with `items` / `total` / `page` / `perPage` / `totalPages` / `hasNext` / `hasPrevious`.
- Typed exception hierarchy (`InvalidIdentifierException`, `InvalidArgumentException`, `InvalidBindException`, `InvalidStateException`, `UnsafeOperationException`) all extending `AppException` — pattern-match the one you care about, catch `AppException` to handle them all.
- Optional automatic `created_at` / `updated_at` columns via `withCreatedAt()` / `withUpdatedAt()`.
- Spring Boot auto-configuration: drop the jar on the classpath, get a `DB` bean.

## Requirements

| | |
|-|-|
| Java | 21+ |
| Spring Framework | 6.x+ (`spring-jdbc`) |
| Spring Boot | 4.x (only required for auto-configuration) |
| Databases | PostgreSQL, MySQL, MariaDB, H2, SQL Server, Oracle (pick the matching `Dialect`) |

## Installation

### Maven

```xml
<dependency>
    <groupId>com.binwi</groupId>
    <artifactId>binwi-dataflow</artifactId>
    <version>2.0.1</version>
</dependency>
```

### Gradle

```groovy
implementation 'com.binwi:binwi-dataflow:2.0.1'
```

Spring JDBC and Spring Boot autoconfigure are declared as `provided` — your application brings them in via `spring-boot-starter-jdbc` or `spring-boot-starter-data-jpa`.

## Quick start

### With Spring Boot (auto-configured)

```java
@Service
public class UserService {
    private final DB db;

    public UserService(DB db) {
        this.db = db;
    }

    public List<Map<String, Object>> activeUsers() {
        return db.table("users")
                .where("active", true)
                .orderByAsc("name")
                .get();
    }
}
```

The auto-configuration creates a prototype-scoped `DB` bean whenever a `NamedParameterJdbcTemplate` exists in the context. It backs off if you declare your own `DB` bean.

### Without Spring Boot

```java
NamedParameterJdbcTemplate jdbc = new NamedParameterJdbcTemplate(myDataSource);
DB db = new DB(jdbc);                                // GenericDialect (default)
DB pg = new DB(jdbc, PostgresDialect.INSTANCE);      // quoted, case-sensitive
```

## Dialects

DataFlow ships seven dialects under `com.binwi.dataflow.dialect`. Pick one to control
identifier quoting, the allowed identifier shape, and pagination syntax. The default is
`GenericDialect` — fully backward compatible with pre-dialect code.

| Dialect | Quote | Identifier pattern | Pagination |
|---|---|---|---|
| `GenericDialect` (default) | none (bare) | `^[a-z][a-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `PostgresDialect`          | `"col"`     | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `H2Dialect`                | `"col"`     | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `MysqlDialect`             | `` `col` `` | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `MariaDbDialect`           | `` `col` `` | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `OracleDialect`            | `"col"`     | `^[A-Za-z_][A-Za-z0-9_]*$` | `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` |
| `SqlServerDialect`         | `[col]`     | `^[A-Za-z_][A-Za-z0-9_]*$` | `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` (requires `ORDER BY`) |

Under Spring Boot, configure it via a single property:

```yaml
binwi:
  dataflow:
    dialect: postgres   # generic | postgres | mysql | mariadb | h2 | sql_server | oracle
```

Or wire it manually:

```java
DB db = new DB(jdbc, PostgresDialect.INSTANCE);

db.table("Users")
    .select("id", "users.name as displayName")
    .where("active", true)
    .innerJoin("posts", "posts.user_id", "users.id", on -> {})
    .orderByAsc("createdAt")
    .take(10)
    .get();
// SELECT "id", "users"."name" as "displayName" FROM "Users"
//   INNER JOIN "posts" ON "posts"."user_id" = "users"."id"
//   WHERE "active" = :active_1
//   ORDER BY "createdAt" ASC LIMIT :_limit
```

### Schema case-sensitivity

`GenericDialect` rejects anything that isn't lowercase snake_case — the strictest, most
portable shape. Picking any other dialect:

- Relaxes validation to `[A-Za-z_][A-Za-z0-9_]*`, so camelCase / PascalCase identifiers
  pass through.
- Quotes every emitted identifier with the dialect's native delimiter, so case-sensitive
  schemas (e.g. quoted Postgres tables, mixed-case Hibernate-generated schemas) resolve
  correctly.
- Lowercases internal placeholder names (e.g. `where("firstName", "Eve")` emits
  `... = :firstname_1`) so generated SQL always survives Spring's named-parameter parser.

### Overriding the dialect with a custom bean

The auto-configuration backs off if you declare your own `Dialect` bean, which is useful
for multi-datasource setups or to inject a custom subclass via a `@Configuration`:

```java
@Configuration
public class DataflowConfig {
    @Bean
    public Dialect dialect() {
        return PostgresDialect.INSTANCE;
    }
}
```

## Examples

### Filtering

```java
db.table("users")
    .where("active", true)
    .whereIn("role", "admin", "owner")           // varargs
    .whereNotNull("email")
    .get();

// also accepts a Collection:
db.table("users").whereIn("role", List.of("admin", "owner")).get();

// negate / range / correlated existence:
db.table("users").whereNotIn("id", 1L, 2L, 3L).get();
db.table("users").whereBetween("age", 18, 65).get();

db.table("users")
    .whereExists(
        "SELECT 1 FROM posts WHERE posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();

// DISTINCT projection:
db.table("users").distinct().select("role").get();
```

### Predicate groups

```java
db.table("users")
    .where("active", true)
    .groupStart()
        .where("role", "admin")
        .orWhere("score", ">=", 100)
    .groupEnd()
    .get();
```
Renders: `WHERE active = :active_1 AND (role = :role_2 OR score >= :score_3)`.

### Searching

```java
db.table("posts").like("title", "spring", LikeMode.BOTH).get();
// title LIKE '%spring%'
```

`LikeMode` is `LEFT` (`%value`), `RIGHT` (`value%`), or `BOTH` (`%value%`). `%` and `_` in the user-provided value are rejected to prevent unbounded scans.

### Joins

```java
db.table("posts")
    .select("posts.title", "users.name")
    .innerJoin("users", "users.id", "posts.user_id", on -> {})
    .where("posts.published", true)
    .get();
```

Adding more ON conditions:

```java
db.table("users")
    .leftJoin("posts", "posts.user_id", "users.id",
        on -> on.where("published", true)
                .orWhereNull("deleted_at"))
    .get();
```

Column-to-column equality inside the ON:

```java
db.table("orders")
    .leftJoin("invoices", "invoices.order_id", "orders.id",
        on -> on.colEq("invoices.tenant_id", "orders.tenant_id"))
    .get();
```

### Inserts

```java
Long id = db.table("users")
    .withCreatedAt()
    .insert(Map.of(
        "name", "Alice",
        "email", "alice@example.com"
    ));
```

Use `insertWithoutId(...)` when the table has a composite or client-supplied primary key — it skips the generated-key lookup and returns the affected row count:

```java
int rows = db.table("session_audit").insertWithoutId(Map.of(
    "session_id", sessionId,
    "ip", ip,
    "ts", Instant.now()
));
```

Batch:

```java
db.table("posts").insertBatch(List.of(
    Map.of("user_id", 1L, "title", "Hello"),
    Map.of("user_id", 1L, "title", "World")
));
```

### Updates

```java
int updated = db.table("users")
    .where("id", 42L)
    .withUpdatedAt()
    .update(Map.of("name", "Alice 2"));
```

`update()` refuses to run without a `WHERE`. It also refuses joins, group-by, order-by, limit/offset, and any attempt to modify `id`.

### Reading single rows

`first()` returns an `Optional` that is empty when no row matches — both the untyped `Map`-returning form and the typed overloads:

```java
Optional<Map<String, Object>> alice = db.table("users").where("name", "Alice").first();
alice.ifPresent(row -> System.out.println(row.get("email")));

Optional<User> user = db.table("users").where("id", 42L).first(User.class);
User u = user.orElseThrow(() -> new NoSuchElementException("user not found"));
```

### Deletes / counters

```java
db.table("sessions").where("expires_at", "<", now).delete();
db.table("users").where("id", 42L).increment("score", 10L);
db.table("users").where("id", 42L).decrement("lives", 1L);
```

### Pagination

```java
Page<Map<String, Object>> page = db.table("users")
    .where("active", true)
    .orderByDesc("created_at")
    .paginate(20, 1);

page.items();        // List<Map<String, Object>> with snake_case keys (immutable)
page.total();        // long total rows ignoring LIMIT/OFFSET
page.page();         // 1
page.perPage();      // 20
page.totalPages();   // 7  (computed)
page.hasNext();      // true
page.hasPrevious();  // false

for (Map<String, Object> row : page) {  // Page<T> implements Iterable<T>
    // ...
}
```

`Page<T>` is a typed `record` defined in `com.binwi.dataflow.Page`. The `items` list is defensively copied and unmodifiable.

### Typed result mapping

Pass a record or POJO class to `get`, `first`, or `paginate` to skip the `Map<String, Object>` step. Spring's `DataClassRowMapper` handles snake_case → camelCase automatically.

```java
public record User(Long id, String name, String email, Boolean active,
                   Integer age, String role, Integer score,
                   Timestamp createdAt, Timestamp updatedAt) {}

List<User> all      = db.table("users").orderByAsc("id").get(User.class);
Optional<User> one  = db.table("users").where("id", 42L).first(User.class);
Page<User> page     = db.table("users").orderByDesc("created_at").paginate(20, 1, User.class);
```

For custom mapping, pass a `RowMapper<T>` instead of a `Class<T>`:

```java
List<String> emails = db.table("users")
    .select("email")
    .get((rs, n) -> rs.getString("email"));
```

### Streaming large result sets

`stream()`, `stream(Class<T>)` and `stream(RowMapper<T>)` return a `java.util.stream.Stream` backed by a live JDBC cursor — useful for iterating millions of rows without loading them into a `List`. The caller **must close** the stream (use try-with-resources):

```java
try (Stream<User> users = db.table("users")
        .where("active", true)
        .orderByAsc("id")
        .stream(User.class)) {
    users.forEach(this::process);
}
```

### Raw escape hatches with safe binds

```java
db.table("users")
    .whereRaw("created_at BETWEEN :from AND :to",
        Map.of("from", start, "to", end))
    .get();

db.table("orders")
    .selectRaw("SUM(amount) AS total")
    .groupBy("user_id")
    .havingRaw("SUM(amount) > :min", Map.of("min", 1000))
    .get();

db.table("users")
    .joinRaw("LEFT JOIN posts ON posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

All raw helpers (`selectRaw`, `whereRaw`, `havingRaw`, `joinRaw`, `orderByAscRaw`, `orderByDescRaw`) reject expressions that contain `;`, `--`, `/*`, or `*/`. Bind keys must match `[a-z][a-z0-9_]*`; missing, unused, or conflicting binds throw an `AppException` with the offending key in the message.

## Safety guarantees

- Identifier validation. Every table / column / alias passed to the public API is checked against `^[a-z][a-z0-9_]*$` (with a single optional dot for qualified columns). SQL injection through identifiers is structurally impossible.
- Parameterized values. Values are bound via Spring's `NamedParameterJdbcTemplate` — never interpolated into the SQL string.
- Refused destructive operations:
  - `delete()` without `WHERE` throws `UnsafeOperationException`.
  - `update()` without `WHERE` throws `UnsafeOperationException`.
  - `update()` with joins / group-by / order-by / limit / offset throws `UnsafeOperationException`.
  - `update()` of the `id` column throws `UnsafeOperationException`.
  - `increment()` / `decrement()` without `WHERE` throws `UnsafeOperationException`.
- Like-wildcard guard. `like()` rejects `%` and `_` inside the user-provided value.
- Empty `whereIn(...)` rejected. An empty value list throws `InvalidArgumentException` instead of silently dropping the predicate.
- Consistent validation. `groupBy()`, `orderByAsc()`, `orderByDesc()` all throw `InvalidIdentifierException` on invalid identifiers (rather than silently skipping them).
- Raw expression hardening. All raw helpers reject `;`, `--`, `/*`, `*/` (throwing `InvalidArgumentException`) and validate binds (throwing `InvalidBindException` on missing / unused / conflicting / malformed keys).

## Exception hierarchy

All exceptions extend `com.binwi.dataflow.exception.AppException`:

| Subclass                       | Thrown when                                                                            |
|--------------------------------|----------------------------------------------------------------------------------------|
| `InvalidIdentifierException`   | table/column/alias/operator/joiner fails validation                                    |
| `InvalidArgumentException`     | null/empty input, bad pagination args, mistyped `like` value, forbidden raw tokens     |
| `InvalidBindException`         | raw expression binds are missing, unused, malformed, or conflict with prior values     |
| `InvalidStateException`        | no table set, unbalanced groups, internal SQL composition failure                      |
| `UnsafeOperationException`     | destructive op refused (mass UPDATE/DELETE, id mutation, etc.)                          |

```java
try {
    db.table("users").update(Map.of("score", 0));
} catch (UnsafeOperationException e) {
    // refusing UPDATE without a WHERE clause — recover or log
} catch (AppException e) {
    // any other DataFlow failure
}
```

## Limitations

- Pagination dialects covered: Postgres, MySQL, MariaDB, H2 (`LIMIT/OFFSET`) and
  Oracle / SQL Server (`OFFSET ... ROWS FETCH NEXT ... ROWS ONLY`). SQL Server requires an
  `ORDER BY` whenever `take()` / `skip()` is set; SQLite and pre-12c Oracle are not
  supported out of the box (use `unsafeQuery` if you must).
- Dialect-specific SQL extensions like Postgres `ILIKE` or `NULLS FIRST/LAST` are still
  caller-responsibility — DataFlow does not rewrite them per dialect. Reach for the raw
  escape hatches (`whereRaw`, `havingRaw`, `orderByAscRaw`) when you need them.
- `DB` is mutable per-instance. The Spring Boot auto-config registers it as `@Scope("prototype")` to give each injection point its own builder; if you wire it manually as a singleton, do not share it across threads.
- No transaction helper baked in — use Spring's `@Transactional` / `TransactionTemplate` around `DB` calls as usual.

## Configuration

The Spring Boot auto-configuration activates whenever:

1. `org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate` is on the classpath, AND
2. A bean of that type exists in the context, AND
3. No user-defined `DB` bean exists.

It also exposes a `Dialect` bean driven by `binwi.dataflow.dialect` (defaults to
`generic`). Override either bean to customise wiring — for example to point at a
non-primary `NamedParameterJdbcTemplate` in a multi-datasource setup:

```java
@Configuration
public class DataflowConfig {
    @Bean
    public DB analyticsDb(@Qualifier("analyticsJdbc") NamedParameterJdbcTemplate jdbc) {
        return new DB(jdbc, PostgresDialect.INSTANCE);
    }
}
```

## Development

```bash
mvn test           # run the JUnit + H2 test suite
mvn package        # build the jar locally (also produces sources + javadoc)
mvn -Prelease deploy   # sign + publish to Maven Central
```

Tests run against an in-memory H2 database configured for snake_case identifiers, so the SQL generated by the library is exercised end-to-end.

## License
See [LICENSE](LICENSE).
