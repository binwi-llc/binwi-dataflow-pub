# API Quick Reference

- [Introduction](#introduction)
- [Construction](#construction)
- [Entry Point](#entry-point)
- [Select](#select)
- [Where Predicates](#where-predicates)
- [Grouping](#grouping)
- [Joins](#joins)
- [Group By, Having, Order, Limit](#group-by-having-order-limit)
- [Timestamps and Keys](#timestamps-and-keys)
- [Reading](#reading)
- [Streaming](#streaming)
- [Pagination](#pagination)
- [Writing](#writing)
- [Introspection and Escape Hatches](#introspection-and-escape-hatches)
- [Static Helpers](#static-helpers)

<a name="introduction"></a>
## Introduction

A compact index of the `DB` API, grouped by category. For full explanations and examples,
follow the links to the topic guides.

<a name="construction"></a>
## Construction

| Method | Description |
|--------|-------------|
| `new DB(jdbc)` | Build with the default `GenericDialect`. |
| `new DB(jdbc, dialect)` | Build with a specific `Dialect`. |
| `dialect()` | Returns the `Dialect` this builder renders for. |

See [Installation & Setup](installation-and-setup.md).

<a name="entry-point"></a>
## Entry Point

| Method | Description |
|--------|-------------|
| `table(name)` | Starts a fresh chain against a table. Every query begins here. |

<a name="select"></a>
## Select

| Method | Description |
|--------|-------------|
| `select(cols...)` | Projects specific columns (supports `as` aliases and qualified names). |
| `selectAll(prefix)` | Projects `prefix.*`. |
| `selectRaw(expr[, binds])` | Adds a raw projection expression. |
| `resetSelect()` | Clears the current projection. |
| `distinct()` | Adds `DISTINCT`. |

See [Querying Data](querying-data.md#selecting-columns).

<a name="where-predicates"></a>
## Where Predicates

| Method | Description |
|--------|-------------|
| `where(col, value)` / `where(col, op, value)` | Equality or explicit-operator condition (AND). |
| `orWhere(...)` | Same, joined with OR. |
| `whereIn` / `whereNotIn` (varargs or `Collection`) | Set membership. |
| `orWhereIn` / `orWhereNotIn` | OR variants. |
| `whereBetween(col, low, high)` / `orWhereBetween` | Inclusive range. |
| `whereNull` / `whereNotNull` / `orWhereNull` / `orWhereNotNull` | Null tests. |
| `whereExists` / `whereNotExists` / `orWhereExists` / `orWhereNotExists` | Correlated subquery. |
| `like(col, value, mode)` / `orLike` | Pattern search with `LikeMode`. |
| `whereRaw(expr[, binds])` / `orWhereRaw(expr, binds)` | Raw condition. |

See [Querying Data](querying-data.md#basic-where-clauses).

<a name="grouping"></a>
## Grouping

| Method | Description |
|--------|-------------|
| `groupStart()` / `orGroupStart()` | Opens a parenthesized predicate group (AND / OR). |
| `groupEnd()` | Closes the current group. |

See [Querying Data](querying-data.md#logical-grouping).

<a name="joins"></a>
## Joins

| Method | Description |
|--------|-------------|
| `innerJoin(table, left, right, on)` | INNER JOIN with an ON-condition DSL. |
| `leftJoin(...)` / `rightJoin(...)` | LEFT / RIGHT JOIN. |
| `leftJoinBefore(...)` | LEFT JOIN prepended before existing joins. |
| `joinRaw(sql[, binds])` | Raw join fragment. |

The `on` consumer (`JoinOnBuilder`) supports `where`, `orWhere`, `whereIn`, `whereNull`,
`colEq`, and `groupStart`/`groupEnd`. See [Joins](joins.md).

<a name="group-by-having-order-limit"></a>
## Group By, Having, Order, Limit

| Method | Description |
|--------|-------------|
| `groupBy(cols...)` | Adds a `GROUP BY`. |
| `havingRaw(expr[, binds])` | Adds a `HAVING` condition (combined with AND). |
| `orderByAsc(col[, nullsFirst])` / `orderByDesc(col[, nullsFirst])` | Ordering. |
| `orderByAscRaw(expr)` / `orderByDescRaw(expr)` | Raw ordering. |
| `take(limit)` | Sets `LIMIT` (bound as `:_limit`). |
| `skip(offset)` | Sets `OFFSET` (bound as `:_offset`). |

See [Querying Data](querying-data.md#ordering).

<a name="timestamps-and-keys"></a>
## Timestamps and Keys

| Method | Description |
|--------|-------------|
| `withCreatedAt()` | Auto-populates `created_at` on insert. |
| `withUpdatedAt()` | Auto-populates `updated_at` on insert/update. |
| `withGeneratedKey(col)` | Names the generated-key column for `insert` (default `id`). |

See [Writing Data](writing-data.md#timestamps).

<a name="reading"></a>
## Reading

| Method | Description |
|--------|-------------|
| `get()` | All rows as `List<Map<String, Object>>`. |
| `get(Class)` | All rows mapped to a record/POJO. |
| `get(RowMapper)` | All rows mapped by a custom `RowMapper`. |
| `first()` | First row as `Optional<Map<String, Object>>`. |
| `first(Class)` / `first(RowMapper)` | First row, typed. |
| `count()` | Number of matching rows (or groups). |

See [Querying Data](querying-data.md#retrieving-results) and [Typed Results](typed-results.md).

<a name="streaming"></a>
## Streaming

| Method | Description |
|--------|-------------|
| `stream()` | Cursor-backed stream of `Map<String, Object>`. |
| `stream(Class)` / `stream(RowMapper)` | Typed cursor-backed stream. |

The caller **must close** the stream. See
[Pagination & Streaming](pagination-and-streaming.md#streaming-large-result-sets).

<a name="pagination"></a>
## Pagination

| Method | Description |
|--------|-------------|
| `paginate(perPage, page)` | Returns a `Page<Map<String, Object>>`. |
| `paginate(perPage, page, Class)` / `paginate(perPage, page, RowMapper)` | Typed page. |

`Page<T>` exposes `items()`, `total()`, `page()`, `perPage()`, `totalPages()`, `hasNext()`,
`hasPrevious()`, and implements `Iterable<T>`. See
[Pagination & Streaming](pagination-and-streaming.md).

<a name="writing"></a>
## Writing

| Method | Description |
|--------|-------------|
| `insert(map)` | Inserts a row, returns the generated key. |
| `insertWithoutId(map)` | Inserts without fetching a key, returns affected count. |
| `insertBatch(rows)` | Batch insert, returns per-row counts. |
| `update(map)` | Updates matching rows (requires `WHERE`). |
| `delete()` | Deletes matching rows (requires `WHERE`). |
| `increment(col, value)` / `decrement(col, value)` | Adjusts a numeric column (requires `WHERE`). |

See [Writing Data](writing-data.md).

<a name="introspection-and-escape-hatches"></a>
## Introspection and Escape Hatches

| Method | Description |
|--------|-------------|
| `toSql()` | Renders SQL + bind map for a read query without executing it. |
| `unsafeQuery(sql, binds)` | Runs raw SQL directly, bypassing all safety. |

See [Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

<a name="static-helpers"></a>
## Static Helpers

| Method | Description |
|--------|-------------|
| `DB.isValidName(str)` | Validates an identifier. |
| `DB.isValidName(str, allowAlias)` | Validates, optionally allowing alias syntax. |
| `DB.isValidAliasName(str)` | Validates an alias-shaped identifier. |

See [Dialects & Identifiers](dialects-and-identifiers.md#identifier-validation).
</content>
