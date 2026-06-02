# Best Practices

Practical guidance for using Binwi DataFlow effectively in production applications.

## Builder lifecycle

### Always start with `table()`

Each query begins with `db.table("...")`, which resets all accumulated state:

```java
// Good — two independent queries on same DB instance
List<User> active = db.table("users").where("active", true).get(User.class);
List<Post> posts = db.table("posts").where("published", true).get(Post.class);
```

### Do not share builders across threads

`DB` is mutable and not thread-safe. With Spring Boot auto-configuration it is prototype-scoped — one instance per injection point.

If you manually create a singleton `DB`, use `ObjectProvider<DB>` or create a new instance per operation:

```java
@Service
class ReportService {
    private final ObjectProvider<DB> db;

    void run() {
        DB query = db.getObject();
        query.table("orders").where("status", "open").stream(Order.class)
             .forEach(this::process);
    }
}
```

## Schema naming

### Prefer snake_case with GenericDialect

For new projects, lowercase snake_case table and column names work with the default dialect and port easily:

```
users, order_items, created_at, user_id
```

### Use quoting dialects for legacy schemas

When integrating with Hibernate, JPA, or mixed-case PostgreSQL schemas:

```yaml
binwi:
  dataflow:
    dialect: postgres
```

Then use the exact table/column casing your schema defines.

## Service layer patterns

### One repository-style class per aggregate

```java
@Repository
public class UserRepository {

    private final DB db;

    public UserRepository(DB db) {
        this.db = db;
    }

    public Optional<User> findById(Long id) {
        return db.table("users").where("id", id).first(User.class);
    }

    public List<User> findActive() {
        return db.table("users").where("active", true).orderByAsc("name").get(User.class);
    }

    public Long create(CreateUser cmd) {
        return db.table("users")
                .withCreatedAt()
                .withUpdatedAt()
                .insert(Map.of("name", cmd.name(), "email", cmd.email(), "active", true));
    }

    public boolean deactivate(Long id) {
        return db.table("users").where("id", id).update(Map.of("active", false)) > 0;
    }
}
```

Centralizing table names and common queries reduces duplication and eases schema migrations.

### Keep business logic out of SQL strings

```java
// Good — logic in Java, data in binds
boolean isEligible = age >= 18 && country.equals("US");
if (isEligible) {
    db.table("users").where("id", id).update(Map.of("eligible", true));
}

// Avoid — business rules buried in whereRaw unless necessary
```

## Transactions

Wrap multi-statement writes in `@Transactional`:

```java
@Transactional
public void transfer(Long from, Long to, long points) {
    int debited = db.table("wallets").where("user_id", from).decrement("balance", points);
    if (debited == 0) throw new InsufficientFundsException();
    db.table("wallets").where("user_id", to).increment("balance", points);
}
```

For read-only streaming at scale:

```java
@Transactional(readOnly = true)
public void export(Consumer<User> consumer) {
    try (Stream<User> s = db.table("users").stream(User.class)) {
        s.forEach(consumer);
    }
}
```

## Pagination in HTTP APIs

Use 1-based page numbers consistently with `paginate(perPage, page)`:

```java
public Page<UserDto> list(int page, int size) {
    return db.table("users")
            .where("active", true)
            .orderByAsc("name")
            .paginate(size, page, User.class)
            .map(u -> new UserDto(u.id(), u.name(), u.email()));
}
```

Validate bounds in the controller:

```java
@GetMapping("/users")
Page<UserDto> list(
        @RequestParam(defaultValue = "1") @Min(1) int page,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size) {
    return userService.list(page, size);
}
```

## Performance

### Select only needed columns

```java
// Prefer
db.table("users").select("id", "name").get(UserSummary.class);

// Over
db.table("users").get(User.class);  // SELECT *
```

### Index columns used in WHERE and ORDER BY

Especially important for paginated lists and SQL Server OFFSET/FETCH.

### Use streaming for large exports

```java
try (Stream<User> users = db.table("users").where("active", true).stream(User.class)) {
    users.map(this::toCsvLine).forEach(csvWriter::println);
}
```

### Consider keyset pagination for deep pages

OFFSET-based pagination slows down on large offsets. For "infinite scroll" on big tables:

```java
db.table("posts")
    .where("published", true)
    .where("id", ">", cursor)
    .orderByAsc("id")
    .take(20)
    .get(Post.class);
```

## Testing

### Integration tests with real SQL

DataFlow's own tests use H2 with snake_case schemas. Mirror that pattern:

```java
@SpringBootTest
class UserRepositoryTest {

    @Autowired DB db;

    @Test
    void findsActiveUsers() {
        List<User> users = db.table("users").where("active", true).get(User.class);
        assertThat(users).isNotEmpty();
    }
}
```

### Assert dialect in configuration tests

```java
@Test
void usesPostgresDialect(@Autowired DB db) {
    assertThat(db.dialect()).isInstanceOf(PostgresDialect.class);
}
```

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `db.table("Users")` with GenericDialect | Use `users` or switch to quoting dialect |
| Forgetting `ORDER BY` before pagination on SQL Server | Add `.orderByAsc("id")` |
| `update()` without `where()` | Always filter updates |
| Not closing `stream()` | Use try-with-resources |
| Passing user input as column names | Validate against allowlist in app code; DataFlow rejects invalid identifiers but not "wrong" valid ones |
| Expecting camelCase keys from `get()` | Use `get(User.class)` or access snake_case keys |
| Empty `whereIn()` | Guard in application code before calling DataFlow |

## When to use JPA instead

Consider JPA/Hibernate when you need:

- Entity relationship navigation and lazy loading
- Automatic schema generation / migrations (Flyway/Liquibase still work with DataFlow)
- Detached entity lifecycle management
- Database-vendor query caching at the ORM level

Use DataFlow when you want explicit SQL, lightweight JDBC, or fine control over queries without ORM overhead.

## When to use raw helpers

Reach for `whereRaw` / `selectRaw` when:

- You need database-specific functions (`jsonb`, full-text search, window functions)
- The expression is not a simple column comparison

Reach for `unsafeQuery` only when the entire SQL shape is dynamic and cannot be expressed any other way — and never with user-controlled SQL structure.

## Logging and debugging

Log generated SQL during development by temporarily wrapping execution or using Spring JDBC debug logging:

```properties
logging.level.org.springframework.jdbc.core=DEBUG
```

DataFlow does not include a built-in SQL logger; the SQL is built at execution time inside `get()`, `first()`, etc.

## Migration from 1.x

Key changes in 2.0.0:

| 1.x | 2.0.0 |
|-----|-------|
| `first()` → `Map` (empty on miss) | `first()` → `Optional` |
| `paginate()` → `Map` with camelCase | `paginate()` → `Page<T>` with snake_case maps |
| `insertNoId()` | `insertWithoutId()` returns `int` |
| `query()` | `unsafeQuery()` |
| `like(col, val, "%x%")` | `like(col, val, LikeMode.BOTH)` |

See [CHANGELOG.md](../CHANGELOG.md) for the full list.
