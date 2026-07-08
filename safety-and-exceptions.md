# Safety & Exceptions

- [Introduction](#introduction)
- [The Safety Guarantees](#the-safety-guarantees)
- [Identifier Validation](#identifier-validation)
- [Parameterized Values](#parameterized-values)
- [Refused Destructive Operations](#refused-destructive-operations)
- [Raw Expression Hardening](#raw-expression-hardening)
- [The Exception Hierarchy](#the-exception-hierarchy)
- [Handling Exceptions](#handling-exceptions)

<a name="introduction"></a>
## Introduction

Safety is DataFlow's central design goal. The library is built so that the *default* way of
writing a query is also the *safe* way. This guide catalogs the guarantees the library
makes and the exceptions it throws when you cross a line.

<a name="the-safety-guarantees"></a>
## The Safety Guarantees

At a glance, DataFlow guarantees:

- **Identifier validation** — every table, column, and alias is checked before use.
- **Parameterized values** — values are always bound, never concatenated into SQL.
- **Refused destructive operations** — mass `UPDATE`/`DELETE` and `id` mutation are blocked.
- **Like-wildcard guarding** — `%` and `_` in a `like` value are rejected.
- **Non-empty `whereIn`** — an empty value list is rejected rather than silently dropped.
- **Consistent validation** — `groupBy` / `orderBy` throw on bad identifiers rather than skipping them.
- **Raw expression hardening** — raw fragments reject statement-breaking tokens and validate binds.

The sections below expand on each.

<a name="identifier-validation"></a>
## Identifier Validation

Every table, column, and alias passed to the public API is validated against the active
dialect's identifier pattern (`^[a-z][a-z0-9_]*$` under `GenericDialect`, relaxed to allow
mixed case under the others), with a single optional dot for qualified columns.

Because an invalid identifier never becomes part of a statement, **SQL injection through
identifiers is structurally impossible**. A bad name throws an `InvalidIdentifierException`
before any query runs. See [Dialects & Identifiers](dialects-and-identifiers.md) for the
exact rules.

<a name="parameterized-values"></a>
## Parameterized Values

Every value — `where` operands, `IN` lists, `BETWEEN` bounds, insert/update data, even
`LIMIT` and `OFFSET` — is bound through Spring's `NamedParameterJdbcTemplate` as a named
parameter. Values are never interpolated into the SQL string. This holds for raw escape
hatches too, where binds are supplied and validated separately from the SQL fragment.

<a name="refused-destructive-operations"></a>
## Refused Destructive Operations

DataFlow refuses operations that are almost always a mistake:

- `delete()` without a `WHERE` clause → `UnsafeOperationException`.
- `update()` without a `WHERE` clause → `UnsafeOperationException`.
- `update()` combined with a join, `GROUP BY`, `ORDER BY`, `LIMIT`, or `OFFSET` → `UnsafeOperationException`.
- `update()` attempting to modify the `id` column → `UnsafeOperationException`.
- `increment()` / `decrement()` without a `WHERE` clause → `UnsafeOperationException`.

Additional value-level guards:

- **Like-wildcard guard** — `like()` rejects `%` and `_` in the user-provided value to
  prevent unbounded scans.
- **Empty `whereIn`** — an empty value list throws `InvalidArgumentException` instead of
  silently dropping the predicate.
- **Consistent validation** — `groupBy()`, `orderByAsc()`, and `orderByDesc()` throw
  `InvalidIdentifierException` on invalid identifiers rather than skipping them silently.

<a name="raw-expression-hardening"></a>
## Raw Expression Hardening

All raw helpers (`selectRaw`, `whereRaw`, `havingRaw`, `joinRaw`, `orderByAscRaw`,
`orderByDescRaw`, and the `whereExists` family) reject expressions containing `;`, `--`,
`/*`, or `*/`, throwing `InvalidArgumentException`. Their binds are validated for presence,
usage, format, and conflicts, throwing `InvalidBindException` on any violation. See
[Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

<a name="the-exception-hierarchy"></a>
## The Exception Hierarchy

All DataFlow exceptions extend `com.binwi.dataflow.exception.AppException`, which is
unchecked. This lets you catch a specific failure when you care, or `AppException` to handle
them all:

| Exception | Thrown when |
|-----------|-------------|
| `InvalidIdentifierException` | a table/column/alias/operator/joiner fails validation |
| `InvalidArgumentException` | null/empty input, bad pagination args, a mistyped `like` value, or a forbidden raw token |
| `InvalidBindException` | raw expression binds are missing, unused, malformed, or conflicting |
| `InvalidStateException` | no table set, unbalanced predicate groups, or an internal composition failure |
| `UnsafeOperationException` | a destructive operation is refused (mass `UPDATE`/`DELETE`, `id` mutation, etc.) |

<a name="handling-exceptions"></a>
## Handling Exceptions

Pattern-match the exception you care about, and fall back to `AppException` for the rest:

```java
try {
    db.table("users").update(Map.of("score", 0));
} catch (UnsafeOperationException e) {
    // refused UPDATE without a WHERE clause — recover or log
    log.warn("Refused unsafe update: {}", e.getMessage());
} catch (AppException e) {
    // any other DataFlow failure
    throw new IllegalStateException("query failed", e);
}
```

Because every exception message names the offending input — the invalid column, the missing
bind, the reason a mutation was refused — failures are self-explanatory in logs and stack
traces.
</content>
