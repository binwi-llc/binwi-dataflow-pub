# Raw SQL Escape Hatches

- [Introduction](#introduction)
- [The Raw Helpers](#the-raw-helpers)
- [Forbidden Tokens](#forbidden-tokens)
- [Bind Validation](#bind-validation)
- [Raw Select and Having](#raw-select-and-having)
- [Raw Where and Exists](#raw-where-and-exists)
- [Raw Joins](#raw-joins)
- [Raw Ordering](#raw-ordering)
- [The Unsafe Query](#the-unsafe-query)

<a name="introduction"></a>
## Introduction

The fluent API covers the common cases, but SQL is vast. When you need an aggregate, a
window function, a database-specific operator, or a subquery the builder can't express,
DataFlow gives you *raw escape hatches* — methods that accept a SQL fragment plus a bind
map. These are safer than they sound: DataFlow still rejects statement-breaking tokens and
validates every bind before running the query.

<a name="the-raw-helpers"></a>
## The Raw Helpers

Every raw helper follows the same pattern — a SQL fragment, and (where values are involved)
a `Map` of named binds:

| Method | Adds to |
|--------|---------|
| `selectRaw(expr[, binds])` | the `SELECT` list |
| `whereRaw(expr[, binds])` / `orWhereRaw(expr, binds)` | the `WHERE` clause |
| `havingRaw(expr[, binds])` | the `HAVING` clause |
| `joinRaw(sql[, binds])` | the join list |
| `orderByAscRaw(expr)` / `orderByDescRaw(expr)` | the `ORDER BY` clause |
| `whereExists` / `whereNotExists` (and `or` variants) | a correlated subquery |

<a name="forbidden-tokens"></a>
## Forbidden Tokens

To prevent a raw fragment from breaking out of its clause or smuggling in a second
statement, all raw helpers reject expressions containing any of:

- `;` — statement terminator
- `--` — line comment
- `/*` or `*/` — block comment delimiters

A fragment containing one of these throws an `InvalidArgumentException`. This guard is not a
substitute for care — it is a backstop. **Never build a raw fragment by concatenating
untrusted input.** Put user data in binds instead.

<a name="bind-validation"></a>
## Bind Validation

Values in raw fragments go through named binds, exactly like the rest of the library. When
you provide binds, DataFlow validates them against the placeholders it finds in the
expression:

- Every `:placeholder` in the expression must have a matching bind.
- Every bind you supply must be referenced by the expression (no unused binds).
- Bind keys must match `[a-z][a-z0-9_]*`.
- A bind must not conflict with a value already registered under the same key.

Any violation throws an `InvalidBindException` naming the offending key. This is what makes
the escape hatches safe: values never touch the SQL string — they're bound.

```java
db.table("users")
    .whereRaw("created_at BETWEEN :from AND :to",
        Map.of("from", start, "to", end))
    .get();
```

<a name="raw-select-and-having"></a>
## Raw Select and Having

`selectRaw` adds an arbitrary projection expression — aggregates, computed columns, and the
like. `havingRaw` filters grouped results:

```java
db.table("orders")
    .selectRaw("SUM(amount) AS total")
    .groupBy("user_id")
    .havingRaw("SUM(amount) > :min", Map.of("min", 1000))
    .get();
```

Multiple `havingRaw` calls are combined with `AND`.

<a name="raw-where-and-exists"></a>
## Raw Where and Exists

`whereRaw` wraps your expression in parentheses and appends it to the `WHERE` clause. Use
`orWhereRaw` to join with `OR`:

```java
db.table("users")
    .where("active", true)
    .orWhereRaw("last_login > :cutoff", Map.of("cutoff", cutoff))
    .get();
```

For correlated subqueries, `whereExists` / `whereNotExists` accept a subquery and its binds
and are subject to the same token and bind rules:

```java
db.table("users")
    .whereExists(
        "SELECT 1 FROM posts WHERE posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

<a name="raw-joins"></a>
## Raw Joins

`joinRaw` takes a complete join fragment. It's the escape hatch for lateral joins, joins to
subqueries, or any join shape the structured API can't produce:

```java
db.table("users")
    .joinRaw("LEFT JOIN posts ON posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

See [Joins](joins.md#raw-joins) for more.

<a name="raw-ordering"></a>
## Raw Ordering

`orderByAscRaw` and `orderByDescRaw` accept an ordering expression rather than a plain
column — useful for ordering by a function or a `CASE`:

```java
db.table("users").orderByDescRaw("COALESCE(last_login, created_at)").get();
```

These validate against forbidden tokens but take no binds.

<a name="the-unsafe-query"></a>
## The Unsafe Query

When nothing else fits — a `UNION`, a CTE, a fully hand-written statement — `unsafeQuery`
passes your SQL and bind map straight to `NamedParameterJdbcTemplate`, bypassing **every**
safety guarantee in the library:

```java
List<Map<String, Object>> rows = db.unsafeQuery(
        "SELECT * FROM users WHERE id = :id",
        Map.of("id", 42L));
```

> [!WARNING]
> `unsafeQuery` performs no identifier validation, no token filtering, and no bind checking.
> You are fully responsible for ensuring the SQL cannot be influenced by untrusted input.
> Treat each call site like a piece of dynamic SQL and prefer the fluent API wherever
> possible.
</content>
