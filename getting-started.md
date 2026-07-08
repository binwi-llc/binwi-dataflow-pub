# Getting Started

- [Introduction](#introduction)
- [Add the Dependency](#add-the-dependency)
- [Inject the DB Bean](#inject-the-db-bean)
- [Your First Query](#your-first-query)
- [Reading a Single Row](#reading-a-single-row)
- [Writing Data](#writing-data)
- [Inspecting the Generated SQL](#inspecting-the-generated-sql)
- [Where to Go Next](#where-to-go-next)

<a name="introduction"></a>
## Introduction

This guide gets you from an empty project to a working query in about five minutes. It
assumes you already have a Spring Boot application with a configured `DataSource`. If you
need the full installation details — Gradle, manual wiring, or non-Boot usage — see the
[Installation & Setup](installation-and-setup.md) guide.

<a name="add-the-dependency"></a>
## Add the Dependency

Add DataFlow to your `pom.xml`:

```xml
<dependency>
    <groupId>com.binwi</groupId>
    <artifactId>binwi-dataflow</artifactId>
    <version>2.0.2</version>
</dependency>
```

DataFlow declares Spring JDBC and Spring Boot autoconfigure as `provided`, so your
application supplies them through `spring-boot-starter-jdbc` or
`spring-boot-starter-data-jpa`. As long as a `NamedParameterJdbcTemplate` exists in your
context, DataFlow auto-configures a `DB` bean for you.

<a name="inject-the-db-bean"></a>
## Inject the DB Bean

Inject `DB` through constructor injection, exactly as you would any Spring bean:

```java
@Service
public class UserService {

    private final DB db;

    public UserService(DB db) {
        this.db = db;
    }
}
```

That's all the wiring you need. Every query begins with `db.table("...")`.

<a name="your-first-query"></a>
## Your First Query

Fetch a filtered, ordered, limited list of rows:

```java
List<Map<String, Object>> users = db.table("users")
        .select("id", "name", "email")
        .where("active", true)
        .where("age", ">", 18)
        .orderByDesc("created_at")
        .take(10)
        .get();
```

Each row comes back as a `Map<String, Object>` keyed by column name. The `get()` method
is *terminal* — it runs the query and returns results. Everything before it just builds
up the statement.

> [!NOTE]
> If you omit `select(...)`, DataFlow selects every column (`SELECT *`).

<a name="reading-a-single-row"></a>
## Reading a Single Row

When you expect at most one row, use `first()`. It returns an `Optional` that is empty
when nothing matches, and forces a `LIMIT 1` for you:

```java
Optional<Map<String, Object>> alice = db.table("users")
        .where("name", "Alice")
        .first();

alice.ifPresent(row -> System.out.println(row.get("email")));
```

<a name="writing-data"></a>
## Writing Data

Insert a row and get its generated key back:

```java
Long id = db.table("users")
        .withCreatedAt()
        .insert(Map.of(
                "name", "Alice",
                "email", "alice@example.com"
        ));
```

Update rows that match a constraint:

```java
int updated = db.table("users")
        .where("id", 42L)
        .withUpdatedAt()
        .update(Map.of("name", "Alice 2"));
```

Delete rows:

```java
db.table("sessions")
        .where("expires_at", "<", now)
        .delete();
```

> [!WARNING]
> `update()`, `delete()`, `increment()`, and `decrement()` all refuse to run without a
> `WHERE` clause, throwing an `UnsafeOperationException`. This is deliberate — it prevents
> accidental full-table mutations. See [Safety & Exceptions](safety-and-exceptions.md).

<a name="inspecting-the-generated-sql"></a>
## Inspecting the Generated SQL

You never have to guess what SQL DataFlow produces. Call `toSql()` on any read builder to
get the rendered statement and its bind map without executing it:

```java
SqlAndParams preview = db.table("users")
        .where("active", true)
        .orderByDesc("created_at")
        .take(10)
        .toSql();

System.out.println(preview.sql());     // SELECT * FROM users WHERE active = :active_1 ...
System.out.println(preview.params());  // {active_1=true, _limit=10}
```

This is invaluable for debugging, logging, and writing assertions in tests.

<a name="where-to-go-next"></a>
## Where to Go Next

- [Querying Data](querying-data.md) — the full set of `where` predicates, grouping, and ordering.
- [Typed Results](typed-results.md) — map rows straight into records and POJOs.
- [Dialects & Identifiers](dialects-and-identifiers.md) — target Postgres, MySQL, Oracle, and more.
- [Best Practices](best-practices.md) — thread safety, transactions, and common patterns.
</content>
