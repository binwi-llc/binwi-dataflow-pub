# Dialects & Identifiers

DataFlow uses a pluggable `Dialect` strategy to control identifier quoting, validation rules, and pagination syntax.

## Available dialects

| Dialect class | Spring property | Quote style | Identifier pattern | Pagination |
|---------------|-----------------|-------------|-------------------|------------|
| `GenericDialect` | `generic` (default) | none (bare) | `^[a-z][a-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `PostgresDialect` | `postgres` | `"col"` | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `H2Dialect` | `h2` | `"col"` | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `MysqlDialect` | `mysql` | `` `col` `` | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `MariaDbDialect` | `mariadb` | `` `col` `` | `^[A-Za-z_][A-Za-z0-9_]*$` | `LIMIT ... OFFSET ...` |
| `OracleDialect` | `oracle` | `"col"` | `^[A-Za-z_][A-Za-z0-9_]*$` | `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` |
| `SqlServerDialect` | `sql_server` | `[col]` | `^[A-Za-z_][A-Za-z0-9_]*$` | `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` |

All dialect instances are singletons accessed via `DialectClass.INSTANCE`.

## GenericDialect (default)

Best for:

- Greenfield schemas using lowercase snake_case
- Maximum portability across databases
- Existing DataFlow 1.x codebases (unchanged behaviour)

Example:

```java
db.table("users")
    .select("id", "name")
    .where("active", true)
    .get();
// SELECT id, name FROM users WHERE active = :active_1
```

Invalid identifiers under GenericDialect:

- `Users` — uppercase not allowed
- `user-id` — hyphens not allowed
- `users.id.extra` — only one dot allowed in qualified names

## Quoting dialects (non-generic)

Selecting any non-generic dialect:

1. **Relaxes validation** to `[A-Za-z_][A-Za-z0-9_]*` — camelCase and PascalCase pass
2. **Quotes every identifier** with the dialect's native delimiter
3. **Lowercases bind parameter names** internally (e.g. `firstName` → `:firstname_1`)

Example with PostgreSQL:

```java
DB db = new DB(jdbc, PostgresDialect.INSTANCE);

db.table("Users")
    .select("id", "users.name as displayName")
    .where("active", true)
    .orderByAsc("createdAt")
    .take(10)
    .get();
```

Rendered SQL:

```sql
SELECT "id", "users"."name" as "displayName"
FROM "Users"
WHERE "active" = :active_1
ORDER BY "createdAt" ASC
LIMIT :_limit
```

## Identifier validation rules

Validation is enforced on every table, column, alias, and join identifier passed to the public API.

### Bare identifiers

Must match the active dialect's pattern (see table above).

### Qualified identifiers

A single dot separates table and column: `users.id`, `posts.user_id`.

- Each segment must independently match the bare identifier rule
- No leading/trailing dots
- At most one dot (no `a.b.c`)

### Aliases in SELECT

Use the `column as alias` form:

```java
db.table("users").select("name as display_name").get();
```

Both `name` and `display_name` must be valid identifiers. Under quoting dialects, both are quoted in output.

### SELECT all from a table

```java
db.table("users").selectAll("users").get();
// Generic: users.*
// Postgres: "users".*
```

## Parameter naming

Column names are sanitized into bind suffixes via `Identifiers.toParamSuffix()`:

- Dots become underscores: `users.id` → `users_id`
- Result is lowercased: `firstName` → `firstname`

Combined with a condition counter, placeholders look like `:active_1`, `:users_id_2`, `:age_3_lo` / `:age_3_hi` (for BETWEEN).

Reserved pagination binds:

| Bind | Used by |
|------|---------|
| `:_limit` | `take()`, `first()`, `paginate()` |
| `:_offset` | `skip()`, `paginate()` |

## Pagination by dialect

### LIMIT / OFFSET (Generic, Postgres, MySQL, MariaDB, H2)

```java
db.table("users").orderByAsc("id").take(10).skip(20).get();
// ... ORDER BY id ASC LIMIT :_limit OFFSET :_offset
```

### OFFSET / FETCH (Oracle, SQL Server)

```java
db.table("users").orderByAsc("id").take(10).skip(20).get();
// ... ORDER BY [id] ASC OFFSET :_offset ROWS FETCH NEXT :_limit ROWS ONLY
```

**SQL Server requirement:** `OFFSET/FETCH` requires an `ORDER BY`. Without one, DataFlow throws `UnsafeOperationException` before hitting the database:

```java
// Throws UnsafeOperationException on SqlServerDialect
db.table("users").take(10).get();

// Correct
db.table("users").orderByAsc("id").take(10).get();
```

When only `take()` is set (no `skip()`), Oracle and SQL Server emit `OFFSET 0 ROWS` automatically.

## Choosing the right dialect

```
┌─────────────────────────────────────────────────────────────┐
│  Are all table/column names lowercase snake_case?           │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │ YES                     │ NO
              ▼                         ▼
     GenericDialect (default)    Match your database:
                                 postgres / mysql / etc.
```

## Programmatic dialect access

```java
Dialect dialect = db.dialect();
Pattern pattern = dialect.identifierPattern();
boolean quoted = dialect.caseSensitive();  // false only for GenericDialect
```

## Validation helpers

`com.binwi.dataflow.Identifiers` exposes static checks (defaulting to GenericDialect):

```java
Identifiers.isValid("users");              // true (generic)
Identifiers.isValid("Users");              // false (generic), true (postgres)
Identifiers.isValid("users.id");           // true
Identifiers.isValid("name as alias");      // false — use isValid(str, true)
Identifiers.isValid("name as alias", true); // true
Identifiers.isPlainName("from");           // bind key validation
```

Static methods on `DB` (`isValidName`, etc.) delegate to `Identifiers` for backward compatibility.

## Custom dialects

Implement the sealed `Dialect` interface (same module only — it is sealed). For application code, pick the closest built-in dialect and use raw escape hatches for database-specific extensions (e.g. Postgres `ILIKE` via `where("col", "ilike", value)` — `ilike` is an allowed operator).

## Limitations

- SQLite has no built-in dialect
- Pre-12c Oracle (no `OFFSET/FETCH`) is not supported out of the box
- Dialect-specific features like `NULLS FIRST/LAST` are emitted when you use `orderByAsc(col, nullsFirst)` but are not rewritten per database — verify your target DB supports the clause
