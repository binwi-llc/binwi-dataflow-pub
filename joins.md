# Joins

- [Introduction](#introduction)
- [Basic Joins](#basic-joins)
- [The ON-Condition DSL](#the-on-condition-dsl)
- [Column-to-Column Equality](#column-to-column-equality)
- [Grouping ON Conditions](#grouping-on-conditions)
- [Join Ordering](#join-ordering)
- [Raw Joins](#raw-joins)
- [Selecting from Joined Tables](#selecting-from-joined-tables)

<a name="introduction"></a>
## Introduction

DataFlow supports `LEFT`, `RIGHT`, and `INNER` joins with a dedicated DSL for expressing
additional ON conditions. Every join column is validated, and every value in an ON
condition is bound as a named parameter — the same safety guarantees that apply everywhere
else.

<a name="basic-joins"></a>
## Basic Joins

Each join method takes the table to join, the two columns to match, and a `Consumer` that
configures extra ON conditions. When you only need the primary equality, pass an empty
lambda:

```java
db.table("posts")
    .select("posts.title", "users.name")
    .innerJoin("users", "users.id", "posts.user_id", on -> {})
    .where("posts.published", true)
    .get();
```

The three join methods are:

- `innerJoin(table, leftColumn, rightColumn, on)`
- `leftJoin(table, leftColumn, rightColumn, on)`
- `rightJoin(table, leftColumn, rightColumn, on)`

The first column argument refers to the joined table, and the second to the base table (or
a fully qualified column). DataFlow qualifies unqualified columns for you.

<a name="the-on-condition-dsl"></a>
## The ON-Condition DSL

The `on` consumer receives a `JoinOnBuilder`, which mirrors the `where` API. Any conditions
you add are appended to the join's `ON` clause with `AND`:

```java
db.table("users")
    .leftJoin("posts", "posts.user_id", "users.id",
        on -> on.where("published", true)
                .orWhereNull("deleted_at"))
    .get();
```

The builder supports `where`, `orWhere`, `whereNull`, `whereNotNull`, `whereIn`, and their
variants — values are bound as parameters just like top-level predicates.

<a name="column-to-column-equality"></a>
## Column-to-Column Equality

Sometimes an ON condition compares two columns rather than a column and a value. Use
`colEq` for that — it renders a column-to-column equality without binding a parameter:

```java
db.table("orders")
    .leftJoin("invoices", "invoices.order_id", "orders.id",
        on -> on.colEq("invoices.tenant_id", "orders.tenant_id"))
    .get();
// ... ON invoices.order_id = orders.id AND invoices.tenant_id = orders.tenant_id
```

<a name="grouping-on-conditions"></a>
## Grouping ON Conditions

The `JoinOnBuilder` supports the same parenthesized grouping as the top-level query builder,
via `groupStart()` / `orGroupStart()` / `groupEnd()`:

```java
db.table("users")
    .leftJoin("posts", "posts.user_id", "users.id",
        on -> on.groupStart()
                    .where("published", true)
                    .orWhereNull("deleted_at")
                .groupEnd())
    .get();
```

As with top-level groups, every group must be balanced or an `InvalidStateException` is
thrown when the SQL is rendered.

<a name="join-ordering"></a>
## Join Ordering

Joins are normally appended in the order you call them. Occasionally you need a join to
appear *before* the others — for example when a later join depends on an alias introduced
earlier. Use `leftJoinBefore` to prepend the join to the list:

```java
db.table("orders")
    .leftJoin("invoices", "invoices.order_id", "orders.id", on -> {})
    .leftJoinBefore("customers", "customers.id", "orders.customer_id", on -> {});
// The customers join is rendered first.
```

<a name="raw-joins"></a>
## Raw Joins

When the structured API cannot express a join — a lateral join, a join to a subquery, or
database-specific syntax — drop to `joinRaw`. You supply the full join fragment and an
optional bind map:

```java
db.table("users")
    .joinRaw("LEFT JOIN posts ON posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

Raw joins are validated like every other raw expression: they reject `;`, `--`, `/*`, and
`*/`, and their binds must be present and well-formed. See
[Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

<a name="selecting-from-joined-tables"></a>
## Selecting from Joined Tables

With joins in play, qualify your selected columns so the database knows which table each
belongs to:

```java
db.table("posts")
    .select("posts.title", "posts.created_at", "users.name as author")
    .innerJoin("users", "users.id", "posts.user_id", on -> {})
    .orderByDesc("posts.created_at")
    .get();
```

To pull every column from a joined table, use `selectAll("users")`, which renders
`users.*`.
</content>
