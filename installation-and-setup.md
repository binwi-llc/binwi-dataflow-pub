# Installation & Setup

- [Requirements](#requirements)
- [Installing with Maven](#installing-with-maven)
- [Installing with Gradle](#installing-with-gradle)
- [Provided Dependencies](#provided-dependencies)
- [Setup with Spring Boot](#setup-with-spring-boot)
- [Setup without Spring Boot](#setup-without-spring-boot)
- [Choosing a Dialect](#choosing-a-dialect)
- [Verifying Your Setup](#verifying-your-setup)

<a name="requirements"></a>
## Requirements

DataFlow targets modern Java and Spring. Make sure your project meets these minimums:

| | |
|-|-|
| Java | 21+ |
| Spring Framework | 6.x+ (`spring-jdbc`) |
| Spring Boot | 4.x (only required for auto-configuration) |
| Databases | PostgreSQL, MySQL, MariaDB, H2, SQL Server, Oracle |

Spring Boot is optional. DataFlow works with plain Spring JDBC — Boot only adds the
convenience of auto-configuring the `DB` bean for you.

<a name="installing-with-maven"></a>
## Installing with Maven

```xml
<dependency>
    <groupId>com.binwi</groupId>
    <artifactId>binwi-dataflow</artifactId>
    <version>2.0.2</version>
</dependency>
```

<a name="installing-with-gradle"></a>
## Installing with Gradle

```groovy
implementation 'com.binwi:binwi-dataflow:2.0.2'
```

<a name="provided-dependencies"></a>
## Provided Dependencies

DataFlow declares its Spring dependencies with `provided` scope:

- `org.springframework:spring-jdbc`
- `org.springframework.boot:spring-boot-autoconfigure`

This means DataFlow does **not** pull a specific Spring version into your build. Instead,
your application brings Spring in transitively — typically through
`spring-boot-starter-jdbc` or `spring-boot-starter-data-jpa`. This keeps DataFlow aligned
with whatever Spring version your application already uses.

> [!NOTE]
> If you are not using Spring Boot, add `spring-jdbc` to your build explicitly. It is the
> only hard runtime requirement.

<a name="setup-with-spring-boot"></a>
## Setup with Spring Boot

With Spring Boot, there is nothing to configure. The auto-configuration registers a `DB`
bean whenever a `NamedParameterJdbcTemplate` is present in the application context:

```java
@Service
public class ReportService {

    private final DB db;

    public ReportService(DB db) {
        this.db = db;
    }
}
```

The auto-configuration registers `DB` as a prototype-scoped bean and backs off entirely if
you declare your own `DB` bean. For the full details — properties, custom beans, and
multi-datasource setups — see [Spring Boot Integration](spring-boot-integration.md).

<a name="setup-without-spring-boot"></a>
## Setup without Spring Boot

Outside of Boot, construct `DB` directly from a `NamedParameterJdbcTemplate`:

```java
NamedParameterJdbcTemplate jdbc = new NamedParameterJdbcTemplate(myDataSource);

DB db = new DB(jdbc);                              // GenericDialect (default)
DB pg = new DB(jdbc, PostgresDialect.INSTANCE);    // quoted, case-sensitive
```

Both constructor arguments are validated — passing a `null` template or dialect throws an
`IllegalArgumentException` immediately.

<a name="choosing-a-dialect"></a>
## Choosing a Dialect

DataFlow defaults to the `GenericDialect`, which emits bare, lowercase identifiers and is
fully portable. If your schema uses mixed-case identifiers, or you want database-native
quoting and pagination, pass a specific dialect:

```java
DB db = new DB(jdbc, PostgresDialect.INSTANCE);
```

The available dialects are `GenericDialect`, `PostgresDialect`, `H2Dialect`,
`MysqlDialect`, `MariaDbDialect`, `SqlServerDialect`, and `OracleDialect`. See
[Dialects & Identifiers](dialects-and-identifiers.md) for how each one affects quoting,
identifier validation, and pagination syntax.

<a name="verifying-your-setup"></a>
## Verifying Your Setup

The quickest way to confirm everything is wired correctly is to render a query without
touching the database:

```java
SqlAndParams preview = db.table("users").where("active", true).toSql();
System.out.println(preview.sql());
```

If that prints a `SELECT` statement, DataFlow is installed and configured. From here, head
to [Getting Started](getting-started.md) or [Querying Data](querying-data.md).
</content>
