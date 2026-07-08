# Querying Data

- [Introduction](#introduction)
- [Selecting Columns](#selecting-columns)
- [Distinct](#distinct)
- [Basic Where Clauses](#basic-where-clauses)
- [Or Where Clauses](#or-where-clauses)
- [Where In and Not In](#where-in-and-not-in)
- [Where Between](#where-between)
- [Where Null](#where-null)
- [Where Exists](#where-exists)
- [Like Searches](#like-searches)
- [Logical Grouping](#logical-grouping)
- [Grouping and Having](#grouping-and-having)
- [Ordering](#ordering)
- [Limit and Offset](#limit-and-offset)
- [Retrieving Results](#retrieving-results)
- [Counting](#counting)

<a name="introduction"></a>
## Introduction

The query builder is the heart of DataFlow. Every read starts with `db.table(...)` and
chains constraints until you call a terminal method like `get()`, `first()`, or `count()`.
This guide covers the full set of `SELECT`-side methods.

Throughout, remember two things: every value you pass is bound as a named parameter (never
concatenated), and every identifier is validated against the active dialect before it
reaches the database.

<a name="selecting-columns"></a>
## Selecting Columns

By default a query selects all columns. Pass column names to `select(...)` to narrow the
projection:

```java
db.table("users").select("id", "name", "email").get();
```

You can alias columns with `as` and qualify them with a table prefix:

```java
db.table("users").select("id", "users.name as displayName").get();
```

To select every column of a specific table (useful with joins), use `selectAll`:

```java
db.table("users").selectAll("users").get();   // users.*
```

If you have already added columns and want to start over, call `resetSelect()`:

```java
db.table("users").select("id").resetSelect().select("email").get(); // selects only email
```

<a name="distinct"></a>
## Distinct

Add `DISTINCT` to the projection with `distinct()`:

```java
db.table("users").distinct().select("role").get();
// SELECT DISTINCT role FROM users
```

<a name="basic-where-clauses"></a>
## Basic Where Clauses

The `where` method comes in two forms. The two-argument form implies equality:

```java
db.table("users").where("active", true).get();
// WHERE active = :active_1
```

The three-argument form takes an explicit operator:

```java
db.table("users").where("age", ">", 18).get();
// WHERE age > :age_1
```

Chained `where` calls are combined with `AND`:

```java
db.table("users")
    .where("active", true)
    .where("age", ">", 18)
    .get();
// WHERE active = :active_1 AND age > :age_2
```

Supported operators include `=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`, and `like`. For `IN`,
`NOT IN`, and `BETWEEN`, use the dedicated methods below — passing those operators to
`where` throws an `InvalidIdentifierException` pointing you to the right method.

<a name="or-where-clauses"></a>
## Or Where Clauses

Every `where` variant has an `orWhere` counterpart that joins with `OR` instead of `AND`:

```java
db.table("users")
    .where("role", "admin")
    .orWhere("score", ">=", 100)
    .get();
// WHERE role = :role_1 OR score >= :score_2
```

<a name="where-in-and-not-in"></a>
## Where In and Not In

Use `whereIn` and `whereNotIn` for set membership. Both accept varargs or a `Collection`:

```java
db.table("users").whereIn("role", "admin", "owner").get();           // varargs
db.table("users").whereIn("role", List.of("admin", "owner")).get();  // Collection

db.table("users").whereNotIn("id", 1L, 2L, 3L).get();
```

The `or` variants — `orWhereIn` and `orWhereNotIn` — are available too.

> [!WARNING]
> An empty value list is rejected with an `InvalidArgumentException` rather than silently
> producing an always-false or always-true predicate. Guard your collections before calling.

<a name="where-between"></a>
## Where Between

`whereBetween` renders an inclusive `BETWEEN ... AND ...` with both bounds bound as
parameters:

```java
db.table("users").whereBetween("age", 18, 65).get();
// WHERE age BETWEEN :age_1_lo AND :age_1_hi
```

An `orWhereBetween` variant is available. Neither bound may be `null`.

<a name="where-null"></a>
## Where Null

Test for `NULL` and `NOT NULL` with dedicated methods — they render `IS NULL` / `IS NOT
NULL` without binding a value:

```java
db.table("users").whereNotNull("email").get();
db.table("users").whereNull("deleted_at").get();
```

The `or` variants are `orWhereNull` and `orWhereNotNull`.

<a name="where-exists"></a>
## Where Exists

Correlate against a subquery with `whereExists` and `whereNotExists`. You supply the
subquery SQL and a bind map:

```java
db.table("users")
    .whereExists(
        "SELECT 1 FROM posts WHERE posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

The subquery is treated as a raw expression: it is checked for forbidden tokens (`;`, `--`,
`/*`, `*/`) and its binds are validated. See
[Raw SQL Escape Hatches](raw-sql-escape-hatches.md) for the rules. The `or` variants are
`orWhereExists` and `orWhereNotExists`.

<a name="like-searches"></a>
## Like Searches

`like` performs a pattern search using a `LikeMode` to control wildcard placement:

```java
db.table("posts").like("title", "spring", LikeMode.BOTH).get();
// title LIKE '%spring%'  (value bound as a parameter)
```

`LikeMode` has three values:

| Mode | Pattern | Matches |
|------|---------|---------|
| `LikeMode.LEFT` | `%value` | values *ending* with the term |
| `LikeMode.RIGHT` | `value%` | values *starting* with the term |
| `LikeMode.BOTH` | `%value%` | values *containing* the term |

The wildcards are added by DataFlow around the bound value. To prevent unbounded scans, the
user-provided value itself must not contain `%` or `_` — doing so throws an
`InvalidArgumentException`. An `orLike` variant is available.

<a name="logical-grouping"></a>
## Logical Grouping

To build parenthesized predicate groups, wrap conditions between `groupStart()` and
`groupEnd()`:

```java
db.table("users")
    .where("active", true)
    .groupStart()
        .where("role", "admin")
        .orWhere("score", ">=", 100)
    .groupEnd()
    .get();
// WHERE active = :active_1 AND (role = :role_2 OR score >= :score_3)
```

Use `orGroupStart()` to join the group with `OR` instead of `AND`. Groups may be nested.

> [!NOTE]
> Every `groupStart()` / `orGroupStart()` must be balanced by a `groupEnd()`. An unbalanced
> group throws an `InvalidStateException` when the query is rendered.

<a name="grouping-and-having"></a>
## Grouping and Having

Aggregate with `groupBy` and filter groups with `havingRaw`:

```java
db.table("orders")
    .selectRaw("SUM(amount) AS total")
    .groupBy("user_id")
    .havingRaw("SUM(amount) > :min", Map.of("min", 1000))
    .get();
```

`groupBy` accepts one or more validated column names. `havingRaw` is a raw expression — it
rejects forbidden tokens and validates its binds. Multiple `havingRaw` calls are combined
with `AND`.

<a name="ordering"></a>
## Ordering

Order results with `orderByAsc` and `orderByDesc`:

```java
db.table("users")
    .orderByAsc("name")
    .orderByDesc("created_at")
    .get();
// ORDER BY name ASC, created_at DESC
```

Both accept an optional `nullsFirst` flag that renders `NULLS FIRST` / `NULLS LAST`:

```java
db.table("users").orderByAsc("last_login", true).get();
// ORDER BY last_login ASC NULLS FIRST
```

For expressions that a plain column name cannot express, use `orderByAscRaw` /
`orderByDescRaw` — see [Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

<a name="limit-and-offset"></a>
## Limit and Offset

Cap and offset results with `take` and `skip`. Both values are bound as parameters
(`:_limit`, `:_offset`) and must be non-negative:

```java
db.table("users")
    .orderByDesc("created_at")
    .take(10)
    .skip(20)
    .get();
```

The rendered pagination syntax depends on the active dialect — see
[Dialects & Identifiers](dialects-and-identifiers.md#pagination-by-dialect). For page-based
navigation with totals, prefer [`paginate()`](pagination-and-streaming.md).

<a name="retrieving-results"></a>
## Retrieving Results

Several terminal methods run the query:

```java
// All matching rows as Map<String, Object>:
List<Map<String, Object>> rows = db.table("users").where("active", true).get();

// The first matching row, or Optional.empty():
Optional<Map<String, Object>> row = db.table("users").where("id", 42L).first();
```

To map rows directly into records or POJOs, pass a class to `get` or `first` — see
[Typed Results](typed-results.md). To iterate very large result sets lazily, use `stream()`
— see [Pagination & Streaming](pagination-and-streaming.md).

<a name="counting"></a>
## Counting

`count()` returns the number of matching rows, ignoring `SELECT`, `ORDER BY`, and
pagination. When a `GROUP BY` is present, it counts the number of groups:

```java
Long active = db.table("users").where("active", true).count();
```
</content>
