# Raw SQL Escape Hatches

DataFlow's fluent API covers most queries. When you need database-specific functions, complex expressions, or non-standard syntax, use the raw helpers — each validates input and binds before execution.

## Available raw methods

| Method | Purpose |
|--------|---------|
| `selectRaw(expr)` / `selectRaw(expr, binds)` | Extra SELECT columns or expressions |
| `whereRaw(expr)` / `whereRaw(expr, binds)` | Custom WHERE predicates |
| `orWhereRaw(expr, binds)` | OR-joined raw WHERE |
| `havingRaw(expr)` / `havingRaw(expr, binds)` | HAVING clauses (with GROUP BY) |
| `joinRaw(sql)` / `joinRaw(sql, binds)` | Full join fragment |
| `orderByAscRaw(expr)` | ORDER BY expression (ASC) |
| `orderByDescRaw(expr)` | ORDER BY expression (DESC) |
| `whereExists(subquery, binds)` | EXISTS (uses same validation) |
| `unsafeQuery(sql, binds)` | **Bypasses all safety** — see below |

## Basic usage

### WHERE with binds

```java
db.table("users")
    .whereRaw("created_at BETWEEN :from AND :to",
        Map.of("from", start, "to", end))
    .get();
```

### SELECT aggregates

```java
db.table("orders")
    .selectRaw("SUM(amount) AS total", null)
    .selectRaw("COUNT(*) AS order_count", null)
    .where("user_id", userId)
    .get();
```

When no placeholders appear in the expression, pass `null` for binds.

### HAVING

```java
db.table("orders")
    .selectRaw("user_id", null)
    .selectRaw("SUM(amount) AS total", null)
    .groupBy("user_id")
    .havingRaw("SUM(amount) > :min", Map.of("min", 1000))
    .get();
```

### JOIN

```java
db.table("users")
    .joinRaw("LEFT JOIN posts ON posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

### ORDER BY expression

```java
db.table("users")
    .orderByDescRaw("COALESCE(last_login, created_at)")
    .get();
```

Raw ORDER BY expressions are **not** quoted — you are responsible for valid SQL.

## Bind validation rules

When an expression contains `:placeholder` tokens, DataFlow validates the bind map:

1. **Every placeholder must have a matching bind key**
2. **Every bind key must appear as a placeholder in the expression**
3. **Keys must match** `[a-z][a-z0-9_]*` (lowercase snake_case)
4. **Duplicate keys with different values** throw `InvalidBindException`
5. **Reusing the same key with the same value** across clauses is allowed

```java
// Valid
Map.of("from", start, "to", end)

// Invalid — camelCase key
Map.of("fromDate", start)  // InvalidBindException

// Invalid — missing bind
.whereRaw("x = :missing", Map.of())  // InvalidBindException
```

Placeholder pattern in expressions: `:([a-z][a-z0-9_]*)`

## Injection hardening

All raw helpers (except `unsafeQuery`) reject expressions containing:

| Token | Reason |
|-------|--------|
| `;` | Statement terminator |
| `--` | SQL line comment |
| `/*` | Block comment start |
| `*/` | Block comment end |

Violation throws `InvalidArgumentException`.

This is defence-in-depth — binds still go through parameterized execution.

## EXISTS subqueries

`whereExists` / `whereNotExists` share raw validation:

```java
db.table("users")
    .whereExists(
        "SELECT 1 FROM orders o WHERE o.user_id = users.id AND o.status = :status",
        Map.of("status", "open"))
    .get();
```

## ILIKE and dialect-specific operators

The fluent `where("col", "ilike", value)` operator is allowed in structured WHERE. For Postgres-specific functions, combine raw and structured:

```java
db.table("users")
    .whereRaw("metadata @> :filter::jsonb", Map.of("filter", jsonString))
    .get();
```

DataFlow does not rewrite dialect-specific syntax — the caller owns SQL correctness.

## unsafeQuery — last resort

```java
List<Map<String, Object>> rows = db.unsafeQuery(
    "SELECT * FROM legacy_view WHERE region = :region",
    Map.of("region", region)
);
```

**Warning:** `unsafeQuery` bypasses:

- Identifier validation
- Raw token rejection
- Bind key validation (Spring still parameterizes `:region`)

Use only when:

- No fluent or raw helper can express the query
- You fully control the SQL string (no user input in SQL structure)
- You accept responsibility for injection safety

Prefer `whereRaw` / `selectRaw` over `unsafeQuery` whenever possible.

## Combining raw and fluent

Raw and fluent clauses share one parameter source:

```java
db.table("orders")
    .select("id", "amount")
    .where("status", "completed")
    .whereRaw("shipped_at >= :ship_date", Map.of("ship_date", cutoff))
    .orderByDesc("shipped_at")
    .get();
```

## Common patterns

### Date ranges

```java
.whereRaw("created_at >= :start AND created_at < :end",
    Map.of("start", dayStart, "end", dayEnd))
```

### Full-text search (Postgres)

```java
.whereRaw("search_vector @@ plainto_tsquery('english', :q)",
    Map.of("q", userQuery))
```

### Conditional dynamic SQL

Build expressions in application code, then pass to raw helpers:

```java
String expr = "priority = :prio AND " + (includeArchived ? "true" : "archived_at IS NULL");
db.table("tasks")
    .whereRaw(expr, Map.of("prio", priority))
    .get();
```

Keep user input in binds, not in the expression structure.

## Error reference

| Exception | Cause |
|-----------|-------|
| `InvalidArgumentException` | Empty expression, forbidden token (`;`, `--`, etc.) |
| `InvalidBindException` | Missing, unused, malformed, or conflicting bind |
| `InvalidIdentifierException` | Invalid joiner in internal raw paths |

See [Safety & Exceptions](safety-and-exceptions.md).
