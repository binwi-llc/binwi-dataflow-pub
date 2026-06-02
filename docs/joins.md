# Joins

DataFlow supports `LEFT`, `RIGHT`, and `INNER` joins with a primary ON equality plus an optional DSL for extra ON conditions.

## Basic join syntax

All structured joins follow the same signature:

```java
db.<joinType>Join(
    joinTable,           // table being joined
    leftTableColumn,     // column on the joined table (or left side of equality)
    rightTableColumn,    // column on the base/other table
    on -> { /* optional extra ON conditions */ }
);
```

### INNER JOIN

```java
db.table("posts")
    .select("posts.title", "users.name")
    .innerJoin("users", "users.id", "posts.user_id", on -> {})
    .where("posts.published", true)
    .get();
```

Rendered (GenericDialect):

```sql
SELECT posts.title, users.name
FROM posts
INNER JOIN users ON users.id = posts.user_id
WHERE posts.published = :published_1
```

### LEFT JOIN

```java
db.table("users")
    .select("users.name", "posts.title")
    .leftJoin("posts", "posts.user_id", "users.id", on -> {})
    .get();
```

Note the column order: `leftJoin(joinTable, leftCol, rightCol, ...)` — the first column reference is on the **joined** table, the second on the **existing** side of the join.

### RIGHT JOIN

```java
db.table("users")
    .rightJoin("departments", "departments.id", "users.department_id", on -> {})
    .get();
```

### LEFT JOIN inserted first

`leftJoinBefore` prepends the join (useful when join order affects optimization or readability):

```java
db.table("orders")
    .leftJoinBefore("customers", "customers.id", "orders.customer_id", on -> {})
    .leftJoin("lines", "lines.order_id", "orders.id", on -> {})
    .get();
```

## Extra ON conditions

Pass a lambda to the last parameter. The lambda receives a `JoinOnBuilder`:

```java
db.table("users")
    .leftJoin("posts", "posts.user_id", "users.id",
        on -> on.where("published", true)
                .orWhereNull("deleted_at"))
    .get();
```

Rendered ON clause:

```sql
LEFT JOIN posts ON posts.user_id = users.id
    AND published = :j1_published_1
    OR deleted_at IS NULL
```

### Column-to-column equality

When the join needs an additional table-to-table match:

```java
db.table("orders")
    .leftJoin("invoices", "invoices.order_id", "orders.id",
        on -> on.colEq("invoices.tenant_id", "orders.tenant_id"))
    .get();
```

`colEq` renders `invoices.tenant_id = orders.tenant_id` without bind parameters.

### Predicate groups in ON

```java
db.table("users")
    .leftJoin("posts", "posts.user_id", "users.id",
        on -> on.groupStart()
                .where("published", true)
                .orWhere("featured", true)
            .groupEnd())
    .get();
```

## JoinOnBuilder API

Available inside the ON lambda:

| Method | Description |
|--------|-------------|
| `where(col, value)` | `col = :bind` |
| `where(col, op, value)` | Comparison with bind |
| `orWhere(...)` | OR-joined condition |
| `whereIn(col, Object[])` | IN with array |
| `orWhereIn(col, Object[])` | OR IN |
| `whereNull(col)` / `whereNotNull(col)` | NULL checks |
| `orWhereNull(col)` / `orWhereNotNull(col)` | OR NULL checks |
| `colEq(left, right)` | Column = column (AND) |
| `colEq(left, right, joiner)` | Column = column with `"and"` or `"or"` |
| `groupStart()` / `orGroupStart()` / `groupEnd()` | Parentheses |

Join ON binds use prefixed names like `:j1_published_1` (join index + column + condition index).

## Raw joins

When the structured API is not enough:

```java
db.table("users")
    .joinRaw("LEFT JOIN posts ON posts.user_id = users.id AND posts.published = :pub",
        Map.of("pub", true))
    .get();
```

Raw joins follow the same injection and bind validation rules as other raw helpers. See [Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

## Multiple joins

Joins are applied in call order (except `leftJoinBefore`):

```java
db.table("orders")
    .innerJoin("customers", "customers.id", "orders.customer_id", on -> {})
    .innerJoin("lines", "lines.order_id", "orders.id", on -> {})
    .innerJoin("products", "products.id", "lines.product_id", on -> {})
    .select("orders.id", "customers.name", "products.sku")
    .get();
```

## Joins and mutations

**UPDATE**, **DELETE**, **increment**, and **decrement** do **not** support joins. Attempting these throws `UnsafeOperationException`.

Use joins only with read operations (`get`, `first`, `count`, `stream`, `paginate`).

## Tips

1. **Qualify columns** after joins to avoid ambiguity: `posts.title`, not `title`
2. **Empty ON lambda** — `on -> {}` is valid when the primary equality is sufficient
3. **Filter in WHERE vs ON** — put row-preserving filters in ON for LEFT JOINs; use WHERE for filters that should exclude rows from the result set
4. **Dialect quoting** — under PostgresDialect, table and column names in join arguments are validated and quoted automatically

## Example: users with published post count

```java
db.table("users")
    .select("users.id", "users.name")
    .selectRaw("COUNT(posts.id) AS post_count", null)
    .leftJoin("posts", "posts.user_id", "users.id",
        on -> on.where("published", true))
    .groupBy("users.id", "users.name")
    .orderByDescRaw("post_count")
    .get();
```

For aggregate selects, use `selectRaw` and `groupBy` together.
