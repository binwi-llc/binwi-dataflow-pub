# Binwi DataFlow — Documentation

Binwi DataFlow is a fluent, parameter-safe SQL query builder for Java applications using Spring's `NamedParameterJdbcTemplate`. These guides cover installation, configuration, and day-to-day usage.

## Quick links

| Guide | Description |
|-------|-------------|
| [Getting Started](getting-started.md) | First query in five minutes |
| [Installation & Setup](installation-and-setup.md) | Maven/Gradle, requirements, manual wiring |
| [Spring Boot Integration](spring-boot-integration.md) | Auto-configuration, properties, custom beans |
| [Dialects & Identifiers](dialects-and-identifiers.md) | Database dialects, quoting, naming rules |
| [Querying Data](querying-data.md) | SELECT, WHERE, filtering, grouping, ordering |
| [Joins](joins.md) | LEFT/RIGHT/INNER joins and ON-condition DSL |
| [Writing Data](writing-data.md) | INSERT, UPDATE, DELETE, increment/decrement |
| [Pagination & Streaming](pagination-and-streaming.md) | `paginate()`, `take()`/`skip()`, cursor streams |
| [Typed Results](typed-results.md) | Records, POJOs, `RowMapper`, snake_case mapping |
| [Raw SQL Escape Hatches](raw-sql-escape-hatches.md) | `whereRaw`, `selectRaw`, bind validation |
| [Safety & Exceptions](safety-and-exceptions.md) | Safety guarantees and exception hierarchy |
| [Best Practices](best-practices.md) | Thread safety, transactions, common patterns |
| [API Quick Reference](api-quick-reference.md) | Method index by category |

## What DataFlow is (and is not)

**DataFlow is:**

- A fluent builder that composes SQL and binds values through Spring's named parameters
- A safety layer that validates identifiers, refuses unsafe mutations, and guards raw expressions
- A thin wrapper around `NamedParameterJdbcTemplate` — not an ORM

**DataFlow is not:**

- A replacement for JPA/Hibernate entity management
- A migration tool or schema generator
- A connection pool or transaction manager (use Spring's `@Transactional` as usual)

## Minimal example

```java
List<Map<String, Object>> users = db.table("users")
        .select("id", "name", "email")
        .where("active", true)
        .where("age", ">", 18)
        .orderByDesc("created_at")
        .take(10)
        .get();
```

## Version

This documentation targets **Binwi DataFlow 2.0.0** (Java 21+, Spring Framework 6.x+, Spring Boot 4.x for auto-configuration).

For release history, see [CHANGELOG.md](../CHANGELOG.md).
