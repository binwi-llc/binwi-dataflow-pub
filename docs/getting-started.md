# Getting Started

This guide walks you through adding Binwi DataFlow to a Spring Boot application and running your first queries.

## Prerequisites

| Requirement | Version |
|-------------|---------|
| Java | 21+ |
| Spring Framework | 6.x+ (`spring-jdbc`) |
| Spring Boot | 4.x (optional; needed for auto-configuration) |
| JDBC driver | For your target database |

Your application must already have a `DataSource` and `NamedParameterJdbcTemplate` on the classpath. The usual way is `spring-boot-starter-jdbc` or `spring-boot-starter-data-jpa`.

## Step 1 — Add the dependency

### Maven

```xml
<dependency>
    <groupId>com.binwi</groupId>
    <artifactId>binwi-dataflow</artifactId>
    <version>2.0.0</version>
</dependency>
```

### Gradle

```groovy
implementation 'com.binwi:binwi-dataflow:2.0.0'
```

Spring JDBC and Spring Boot autoconfigure are `provided` scope — your app supplies them via the starter you already use.

## Step 2 — Configure your dialect (optional)

If your schema uses lowercase snake_case identifiers (e.g. `users`, `created_at`), the default `generic` dialect works out of the box.

If you use mixed-case or quoted identifiers (common with Hibernate on PostgreSQL), set the dialect:

```yaml
# application.yml
binwi:
  dataflow:
    dialect: postgres   # generic | postgres | mysql | mariadb | h2 | sql_server | oracle
```

See [Dialects & Identifiers](dialects-and-identifiers.md) for the full comparison table.

## Step 3 — Inject `DB` and query

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

    public Optional<Map<String, Object>> findByEmail(String email) {
        return db.table("users")
                .where("email", email)
                .first();
    }
}
```

Auto-configuration creates a **prototype-scoped** `DB` bean whenever:

1. `NamedParameterJdbcTemplate` is on the classpath and registered as a bean, and
2. You have **not** declared your own `DB` bean.

Each injection point receives a fresh builder instance. See [Spring Boot Integration](spring-boot-integration.md) for multi-datasource setups.

## Step 4 — Use typed results (recommended)

Define a record matching your columns (snake_case in the database maps to camelCase in Java):

```java
public record User(
        Long id,
        String name,
        String email,
        Boolean active,
        Timestamp createdAt
) {}

List<User> users = db.table("users")
        .where("active", true)
        .orderByAsc("name")
        .get(User.class);
```

## Common next steps

| Task | Guide |
|------|-------|
| Filter with `whereIn`, `whereBetween`, groups | [Querying Data](querying-data.md) |
| Join tables | [Joins](joins.md) |
| Insert or update rows | [Writing Data](writing-data.md) |
| Paginate API responses | [Pagination & Streaming](pagination-and-streaming.md) |
| Wire without Spring Boot | [Installation & Setup](installation-and-setup.md) |

## Without Spring Boot

If you do not use Spring Boot auto-configuration, construct `DB` manually:

```java
NamedParameterJdbcTemplate jdbc = new NamedParameterJdbcTemplate(dataSource);
DB db = new DB(jdbc);                           // GenericDialect (default)
DB pg  = new DB(jdbc, PostgresDialect.INSTANCE);  // quoted identifiers
```

Import dialects from `com.binwi.dataflow.dialect.*`.
