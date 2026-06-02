# Spring Boot Integration

Binwi DataFlow ships Spring Boot auto-configuration that registers a `DB` bean and a `Dialect` bean when conditions are met.

## Activation conditions

Auto-configuration runs when **all** of the following are true:

1. `NamedParameterJdbcTemplate` is on the classpath
2. A `NamedParameterJdbcTemplate` bean exists in the application context
3. No user-defined `DB` bean is present

The configuration class is `com.binwi.dataflow.autoconfigure.BinwiDataflowAutoConfiguration`, registered via:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

## Configuration properties

All properties live under the `binwi.dataflow` prefix.

```yaml
binwi:
  dataflow:
    dialect: postgres
```

| Property | Type | Default | Values |
|----------|------|---------|--------|
| `binwi.dataflow.dialect` | enum | `generic` | `generic`, `postgres`, `mysql`, `mariadb`, `h2`, `sql_server`, `oracle` |

Property names are case-insensitive in YAML (`postgres`, `POSTGRES`, and `Postgres` all map to `POSTGRES`).

### application.properties

```properties
binwi.dataflow.dialect=postgres
```

## Beans registered

| Bean | Scope | Condition |
|------|-------|-----------|
| `Dialect` (`binwiDataflowDialect`) | singleton | `@ConditionalOnMissingBean(Dialect.class)` |
| `DB` (`binwiDataflowDb`) | **prototype** | `@ConditionalOnMissingBean(DB.class)` |

### Why prototype scope?

`DB` accumulates query state (SELECT columns, WHERE clauses, joins) across method calls. A singleton would be unsafe if shared across threads or concurrent requests. Prototype scope gives each injection site its own builder.

**Recommended injection patterns:**

```java
// Constructor injection — new instance per service (typical)
@Service
class UserService {
    private final DB db;
    UserService(DB db) { this.db = db; }
}

// ObjectProvider — fetch a fresh builder per operation
@Service
class ReportService {
    private final ObjectProvider<DB> dbProvider;
    ReportService(ObjectProvider<DB> dbProvider) { this.dbProvider = dbProvider; }

    void runReport() {
        DB db = dbProvider.getObject();
        db.table("orders").where("status", "open").get();
    }
}
```

Do **not** store a shared `DB` instance in a static field or singleton service field that multiple threads mutate concurrently.

## Overriding the dialect

Declare your own `Dialect` bean. Auto-configuration backs off:

```java
@Configuration
public class DataflowConfig {

    @Bean
    public Dialect dialect() {
        return PostgresDialect.INSTANCE;
    }
}
```

The `binwi.dataflow.dialect` property is ignored when a custom `Dialect` bean exists.

## Overriding the DB bean

Declare your own `DB` bean for full control (multi-datasource, custom dialect per datasource):

```java
@Configuration
public class DataflowConfig {

    @Bean
    public DB analyticsDb(
            @Qualifier("analyticsJdbc") NamedParameterJdbcTemplate jdbc) {
        return new DB(jdbc, PostgresDialect.INSTANCE);
    }
}
```

When you declare a `DB` bean, auto-configuration does **not** register its own.

## Multi-datasource example

```java
@Configuration
public class DatabaseConfig {

    @Bean
    @Primary
    public DataSource primaryDataSource() { /* ... */ }

    @Bean
    public DataSource analyticsDataSource() { /* ... */ }

    @Bean
    @Primary
    public NamedParameterJdbcTemplate primaryJdbc(
            @Qualifier("primaryDataSource") DataSource ds) {
        return new NamedParameterJdbcTemplate(ds);
    }

    @Bean
    public NamedParameterJdbcTemplate analyticsJdbc(
            @Qualifier("analyticsDataSource") DataSource ds) {
        return new NamedParameterJdbcTemplate(ds);
    }

    @Bean
    @Primary
    public DB db(NamedParameterJdbcTemplate primaryJdbc, Dialect dialect) {
        return new DB(primaryJdbc, dialect);
    }

    @Bean
    public DB analyticsDb(
            @Qualifier("analyticsJdbc") NamedParameterJdbcTemplate jdbc) {
        return new DB(jdbc, PostgresDialect.INSTANCE);
    }
}
```

Inject with `@Qualifier`:

```java
@Service
class AnalyticsService {
    private final DB analyticsDb;

    AnalyticsService(@Qualifier("analyticsDb") DB analyticsDb) {
        this.analyticsDb = analyticsDb;
    }
}
```

## Transactions

DataFlow does not manage transactions. Wrap service methods with Spring's declarative transactions:

```java
@Service
public class OrderService {

    private final DB db;

    @Transactional
    public void deactivateUser(Long userId) {
        db.table("users").where("id", userId).update(Map.of("active", false));
        db.table("sessions").where("user_id", userId).delete();
    }
}
```

Both operations participate in the same transaction because each `db.table(...)` call starts a fresh query on the same prototype instance within one request — but for clarity, some teams prefer `ObjectProvider<DB>` and a new builder per statement.

## Disabling auto-configuration

If you want full manual control, exclude the auto-configuration:

```java
@SpringBootApplication(exclude = BinwiDataflowAutoConfiguration.class)
public class Application { }
```

Then register `DB` and `Dialect` beans yourself.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| No `DB` bean | No `NamedParameterJdbcTemplate` | Add `spring-boot-starter-jdbc` and configure `DataSource` |
| `InvalidIdentifierException` on table name | Wrong dialect for schema | Set `binwi.dataflow.dialect` or use `GenericDialect` with snake_case |
| Shared state between requests | Singleton `DB` wired manually | Use prototype scope or `ObjectProvider` |
| SQL Server pagination error | `take()`/`skip()` without `ORDER BY` | Add `.orderByAsc("id")` before pagination |
