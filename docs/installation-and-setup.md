# Installation & Setup

## Requirements

| Component | Minimum |
|-----------|---------|
| Java | 21 |
| Maven | 3.8+ (for building from source) |
| Spring Framework | 6.x (`spring-jdbc`) |
| Spring Boot | 4.x (auto-configuration only) |

Supported databases (with matching dialect):

- PostgreSQL — `PostgresDialect`
- MySQL — `MysqlDialect`
- MariaDB — `MariaDbDialect`
- H2 — `H2Dialect`
- Microsoft SQL Server — `SqlServerDialect`
- Oracle — `OracleDialect`
- Any JDBC database — `GenericDialect` (default; `LIMIT/OFFSET` pagination)

## Maven

```xml
<dependency>
    <groupId>com.binwi</groupId>
    <artifactId>binwi-dataflow</artifactId>
    <version>2.0.0</version>
</dependency>
```

Ensure your application also depends on Spring JDBC:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

## Gradle

```groovy
implementation 'com.binwi:binwi-dataflow:2.0.0'
implementation 'org.springframework.boot:spring-boot-starter-jdbc'
```

## Manual wiring (no Spring Boot)

```java
import com.binwi.dataflow.DB;
import com.binwi.dataflow.dialect.PostgresDialect;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;

import javax.sql.DataSource;

// 1. Create NamedParameterJdbcTemplate from your DataSource
NamedParameterJdbcTemplate jdbc = new NamedParameterJdbcTemplate(dataSource);

// 2. Create DB with optional dialect
DB db = new DB(jdbc);                        // GenericDialect
DB db = new DB(jdbc, PostgresDialect.INSTANCE);
```

Pass the `DB` instance to your services however you manage dependencies (manual constructor injection, a `@Configuration` `@Bean`, etc.).

## Building from source

```bash
git clone https://github.com/binwi/binwi-dataflow.git
cd binwi-dataflow
mvn test           # run test suite (H2 in-memory)
mvn package        # build jar + sources + javadoc
mvn -Prelease deploy   # sign and publish (maintainers)
```

## Package structure

| Package | Purpose |
|---------|---------|
| `com.binwi.dataflow` | Core `DB` builder, `Page`, `LikeMode`, `JoinOnBuilder` |
| `com.binwi.dataflow.dialect` | `Dialect` implementations |
| `com.binwi.dataflow.autoconfigure` | Spring Boot auto-configuration |
| `com.binwi.dataflow.exception` | Typed exception hierarchy |

## Choosing a dialect at setup time

| Your schema | Recommended dialect |
|-------------|---------------------|
| Lowercase snake_case (`users`, `created_at`) | `GenericDialect` (default) |
| Mixed-case / quoted (Hibernate, JPA) | Match your database |
| PostgreSQL with `"Users"` table | `PostgresDialect` |
| MySQL with backtick-quoted names | `MysqlDialect` or `MariaDbDialect` |
| SQL Server | `SqlServerDialect` (note: pagination requires `ORDER BY`) |
| Oracle 12c+ | `OracleDialect` |

Configure via Spring Boot property or constructor — details in [Dialects & Identifiers](dialects-and-identifiers.md).

## Verifying the setup

After wiring, confirm the bean and dialect in a test or startup log:

```java
@Autowired
DB db;

@Test
void dialectIsConfigured() {
    assertThat(db.dialect()).isInstanceOf(PostgresDialect.class);
}
```

Or run a trivial query:

```java
long count = db.table("users").count();
```

If `table()` throws `InvalidIdentifierException`, your table name may not match the active dialect's identifier rules.
