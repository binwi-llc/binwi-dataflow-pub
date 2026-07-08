# Binwi DataFlow

- [Introduction](#introduction)
- [What DataFlow Is (and Is Not)](#what-dataflow-is-and-is-not)
- [A First Look](#a-first-look)
- [Documentation](#documentation)
- [Version Compatibility](#version-compatibility)

<a name="introduction"></a>
## Introduction

Binwi DataFlow is a fluent, parameter-safe SQL query builder for Java, built on top
of Spring's `NamedParameterJdbcTemplate`. It gives you an expressive, chainable API
for composing `SELECT`, `INSERT`, `UPDATE`, and `DELETE` statements while keeping
every value bound through named parameters — so your SQL stays readable, auditable,
and safe from injection.

If you have ever enjoyed Laravel's query builder and wished for something similar on
the JVM, DataFlow will feel immediately familiar. You start from a table, chain the
constraints you need, and finish with a terminal method that runs the query:

```java
List<Map<String, Object>> users = db.table("users")
        .select("id", "name", "email")
        .where("active", true)
        .where("age", ">", 18)
        .orderByDesc("created_at")
        .take(10)
        .get();
```

<a name="what-dataflow-is-and-is-not"></a>
## What DataFlow Is (and Is Not)

DataFlow is intentionally small. It composes SQL and binds values — nothing more.

**DataFlow is:**

- A fluent builder that composes SQL and binds values through Spring's named parameters.
- A safety layer that validates identifiers, refuses unsafe mutations, and guards raw expressions.
- A thin, predictable wrapper around `NamedParameterJdbcTemplate`.

**DataFlow is not:**

- An ORM or a replacement for JPA/Hibernate entity management.
- A migration tool or schema generator.
- A connection pool or transaction manager — use Spring's `@Transactional` as usual.

<a name="a-first-look"></a>
## A First Look

Once wired into a Spring Boot application, `DB` is injected like any other bean:

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

Every call to `table(...)` returns a *fresh* builder, so a single `DB` instance is safe
to share as a stateless factory across your application.

<a name="documentation"></a>
## Documentation

These guides walk through DataFlow from installation to advanced usage. Read them in
order, or jump to the topic you need.

| Guide | Description |
|-------|-------------|
| [Getting Started](getting-started.md) | Your first query in five minutes |
| [Installation & Setup](installation-and-setup.md) | Maven/Gradle, requirements, manual wiring |
| [Spring Boot Integration](spring-boot-integration.md) | Auto-configuration, properties, custom beans |
| [Dialects & Identifiers](dialects-and-identifiers.md) | Database dialects, quoting, naming rules |
| [Querying Data](querying-data.md) | `SELECT`, `WHERE`, filtering, grouping, ordering |
| [Joins](joins.md) | `LEFT`/`RIGHT`/`INNER` joins and the ON-condition DSL |
| [Writing Data](writing-data.md) | `INSERT`, `UPDATE`, `DELETE`, increment/decrement |
| [Pagination & Streaming](pagination-and-streaming.md) | `paginate()`, `take()`/`skip()`, cursor streams |
| [Typed Results](typed-results.md) | Records, POJOs, `RowMapper`, snake_case mapping |
| [Raw SQL Escape Hatches](raw-sql-escape-hatches.md) | `whereRaw`, `selectRaw`, bind validation |
| [Safety & Exceptions](safety-and-exceptions.md) | Safety guarantees and the exception hierarchy |
| [Best Practices](best-practices.md) | Thread safety, transactions, common patterns |
| [API Quick Reference](api-quick-reference.md) | Method index by category |

<a name="version-compatibility"></a>
## Version Compatibility

This documentation targets **Binwi DataFlow 2.0.2**.

| | |
|-|-|
| Java | 21+ |
| Spring Framework | 6.x+ (`spring-jdbc`) |
| Spring Boot | 4.x (only required for auto-configuration) |
| Databases | PostgreSQL, MySQL, MariaDB, H2, SQL Server, Oracle |

For release history, see [CHANGELOG.md](../CHANGELOG.md).
</content>
</invoke>
