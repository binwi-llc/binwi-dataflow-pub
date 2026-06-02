# Querying Data

Every query starts with `db.table("table_name")`, which resets all builder state and sets the target table.

## SELECT projection

### All columns (default)

If you call no `select()`, DataFlow renders `SELECT *`:

```java
db.table("users").where("active", true).get();
// SELECT * FROM users WHERE active = :active_1
```

### Specific columns

```java
db.table("users").select("id", "name", "email").get();
```

### Table-qualified columns and aliases

```java
db.table("posts")
    .select("posts.title", "users.name as author_name")
    .innerJoin("users", "users.id", "posts.user_id", on -> {})
    .get();
```

### All columns from a joined table

```java
db.table("posts").selectAll("posts").get();
// SELECT posts.* FROM posts
```

### DISTINCT

```java
db.table("users").distinct().select("role").get();
// SELECT DISTINCT role FROM users
```

### Reset projection

```java
db.table("users")
    .select("id")
    .resetSelect()
    .select("name", "email")
    .get();
```

## WHERE clauses

### Equality and comparisons

```java
db.table("users").where("active", true).get();
db.table("users").where("age", ">", 18).get();
db.table("users").where("role", "!=", "guest").get();
```

Supported operators: `=`, `!=`, `<>`, `>`, `<`, `>=`, `<=`, `like`, `ilike`.

Use dedicated methods instead of operators for these cases:

| Instead of | Use |
|------------|-----|
| `where("col", "in", ...)` | `whereIn("col", ...)` |
| `where("col", "not in", ...)` | `whereNotIn("col", ...)` |
| `where("col", "between", ...)` | `whereBetween("col", low, high)` |

### NULL checks

```java
db.table("users").whereNull("deleted_at").get();
db.table("users").whereNotNull("email").get();
```

### IN / NOT IN

Varargs:

```java
db.table("users").whereIn("role", "admin", "owner").get();
db.table("users").whereNotIn("id", 1L, 2L, 3L).get();
```

Collection:

```java
db.table("users").whereIn("role", List.of("admin", "owner")).get();
```

Empty lists throw `InvalidArgumentException` — DataFlow never silently omits an IN clause.

### BETWEEN

```java
db.table("users").whereBetween("age", 18, 65).get();
// WHERE age BETWEEN :age_1_lo AND :age_1_hi
```

Both bounds must be non-null.

### EXISTS / NOT EXISTS

For correlated subqueries:

```java
db.table("users")
    .whereExists(
        "SELECT 1 FROM posts WHERE posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

Subqueries follow the same safety rules as raw expressions (no `;`, `--`, `/*`, `*/`; binds validated).

### LIKE search

```java
import com.binwi.dataflow.LikeMode;

db.table("posts").like("title", "spring", LikeMode.BOTH).get();
// title LIKE '%spring%'
```

| `LikeMode` | Pattern |
|------------|---------|
| `LEFT` | `%value` |
| `RIGHT` | `value%` |
| `BOTH` | `%value%` |

User input must not contain `%` or `_` — DataFlow rejects wildcards in the search term to prevent unbounded scans and accidental pattern injection.

### OR variants

Every predicate has an `or*` counterpart:

```java
db.table("users")
    .where("active", true)
    .orWhere("role", "admin")
    .get();
// WHERE active = :active_1 OR role = :role_2
```

Available: `orWhere`, `orLike`, `orWhereNull`, `orWhereNotNull`, `orWhereIn`, `orWhereNotIn`, `orWhereBetween`, `orWhereExists`, `orWhereNotExists`, `orWhereRaw`.

## Predicate groups

Parenthesize conditions with `groupStart()` / `groupEnd()`:

```java
db.table("users")
    .where("active", true)
    .groupStart()
        .where("role", "admin")
        .orWhere("score", ">=", 100)
    .groupEnd()
    .get();
```

Rendered: `WHERE active = :active_1 AND (role = :role_2 OR score >= :score_3)`

Start a group with OR:

```java
db.table("users")
    .where("active", true)
    .orGroupStart()
        .where("role", "admin")
        .where("verified", true)
    .groupEnd()
    .get();
```

Unbalanced groups throw `InvalidStateException` at query execution time.

## GROUP BY and HAVING

```java
db.table("orders")
    .selectRaw("user_id", null)
    .selectRaw("SUM(amount) AS total", null)
    .groupBy("user_id")
    .havingRaw("SUM(amount) > :min", Map.of("min", 1000))
    .get();
```

- `groupBy()` requires at least one column
- Invalid column names throw `InvalidIdentifierException`

## ORDER BY

```java
db.table("users").orderByAsc("name").get();
db.table("users").orderByDesc("created_at").get();
```

### NULLS FIRST / LAST

```java
db.table("users").orderByAsc("last_login", true).get();   // NULLS FIRST
db.table("users").orderByDesc("last_login", false).get();  // NULLS LAST
```

Verify your database supports `NULLS FIRST/LAST` before using this in production.

### Raw ORDER BY

For expressions DataFlow does not quote:

```java
db.table("users").orderByAscRaw("LENGTH(name)").get();
```

## LIMIT and OFFSET

```java
db.table("users")
    .orderByDesc("created_at")
    .take(10)    // LIMIT — bound as :_limit
    .skip(20)    // OFFSET — bound as :_offset
    .get();
```

Negative values throw `InvalidArgumentException`.

## Reading results

### List of maps

```java
List<Map<String, Object>> rows = db.table("users").get();
// Keys are snake_case column names as returned by JDBC
```

### First row

```java
Optional<Map<String, Object>> row = db.table("users")
    .where("email", email)
    .first();
```

`first()` sets `LIMIT 1` internally and returns `Optional.empty()` when no row matches.

### Count

```java
long total = db.table("users").where("active", true).count();
```

`count()` runs a separate `SELECT COUNT(*) ...` and does not mutate builder state.

## Execution flow

```
table("users")  →  resets state, sets table
    ↓
select / where / join / orderBy / take  →  accumulates fragments
    ↓
get() / first() / count() / stream()  →  builds SQL + executes via NamedParameterJdbcTemplate
```

Each `table(...)` call starts a completely new query. Reusing the same `DB` instance is safe as long as you always begin with `table()`.

## See also

- [Joins](joins.md) — multi-table queries
- [Pagination & Streaming](pagination-and-streaming.md) — `paginate()`, `stream()`
- [Typed Results](typed-results.md) — `get(User.class)`
- [Raw SQL Escape Hatches](raw-sql-escape-hatches.md) — `whereRaw`, `selectRaw`
