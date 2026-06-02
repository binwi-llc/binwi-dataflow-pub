# API Quick Reference

Complete method index for `com.binwi.dataflow.DB`. All methods return `DB` (fluent) unless noted.

## Entry point

| Method | Description |
|--------|-------------|
| `table(String name)` | Set target table; reset all query state |
| `dialect()` | Returns active `Dialect` (not fluent) |

## SELECT

| Method | Description |
|--------|-------------|
| `select(String... columns)` | Add columns (supports `col as alias`) |
| `selectAll(String prefix)` | Add `prefix.*` |
| `selectRaw(String expr)` | Raw SELECT expression |
| `selectRaw(String expr, Map binds)` | Raw SELECT with binds |
| `resetSelect()` | Clear SELECT list |
| `distinct()` | Add DISTINCT |

## WHERE

| Method | Description |
|--------|-------------|
| `where(String col, Object val)` | `col = val` |
| `where(String col, String op, Object val)` | Comparison |
| `orWhere(...)` | OR variants of above |
| `whereNull(col)` / `whereNotNull(col)` | NULL checks |
| `orWhereNull(col)` / `orWhereNotNull(col)` | OR NULL checks |
| `whereIn(col, Object...)` / `whereIn(col, Collection)` | IN |
| `orWhereIn(...)` | OR IN |
| `whereNotIn(...)` / `orWhereNotIn(...)` | NOT IN |
| `whereBetween(col, low, high)` | BETWEEN |
| `orWhereBetween(...)` | OR BETWEEN |
| `whereExists(subquery, binds)` | EXISTS |
| `whereNotExists(...)` / `orWhereExists(...)` / `orWhereNotExists(...)` | EXISTS variants |
| `like(col, val, LikeMode)` / `orLike(...)` | LIKE with mode |
| `whereRaw(expr)` / `whereRaw(expr, binds)` | Raw predicate |
| `orWhereRaw(expr, binds)` | OR raw predicate |
| `groupStart()` / `orGroupStart()` | Open parenthesis group |
| `groupEnd()` | Close parenthesis group |

## JOIN

| Method | Description |
|--------|-------------|
| `leftJoin(table, leftCol, rightCol, on)` | LEFT JOIN |
| `leftJoinBefore(...)` | LEFT JOIN prepended |
| `rightJoin(...)` | RIGHT JOIN |
| `innerJoin(...)` | INNER JOIN |
| `joinRaw(sql)` / `joinRaw(sql, binds)` | Raw join clause |

## GROUP BY / HAVING / ORDER BY

| Method | Description |
|--------|-------------|
| `groupBy(String... cols)` | GROUP BY |
| `havingRaw(expr)` / `havingRaw(expr, binds)` | HAVING |
| `orderByAsc(col)` | ORDER BY ASC |
| `orderByAsc(col, nullsFirst)` | ORDER BY ASC NULLS FIRST/LAST |
| `orderByAscRaw(expr)` | Raw ORDER BY ASC |
| `orderByDesc(col)` | ORDER BY DESC |
| `orderByDesc(col, nullsFirst)` | ORDER BY DESC NULLS FIRST/LAST |
| `orderByDescRaw(expr)` | Raw ORDER BY DESC |

## Pagination limits

| Method | Description |
|--------|-------------|
| `take(long limit)` | LIMIT / FETCH NEXT (`:_limit`) |
| `skip(long offset)` | OFFSET (`:_offset`) |

## Timestamps (INSERT/UPDATE)

| Method | Description |
|--------|-------------|
| `withCreatedAt()` | Auto-set `created_at` on insert |
| `withUpdatedAt()` | Auto-set `updated_at` on insert/update/increment/decrement |
| `withGeneratedKey(col)` | Generated-key column for `insert()` (default `id`) |
| `toSql()` | Preview SELECT SQL + binds (`SqlAndParams`), no execution |

## Read execution

| Method | Returns |
|--------|---------|
| `get()` | `List<Map<String, Object>>` |
| `get(Class<T>)` | `List<T>` |
| `get(RowMapper<T>)` | `List<T>` |
| `first()` | `Optional<Map<String, Object>>` |
| `first(Class<T>)` | `Optional<T>` |
| `first(RowMapper<T>)` | `Optional<T>` |
| `stream()` | `Stream<Map<String, Object>>` — must close |
| `stream(Class<T>)` | `Stream<T>` — must close |
| `stream(RowMapper<T>)` | `Stream<T>` — must close |
| `count()` | `long` |
| `paginate(perPage, page)` | `Page<Map<String, Object>>` |
| `paginate(perPage, page, Class<T>)` | `Page<T>` |
| `paginate(perPage, page, RowMapper<T>)` | `Page<T>` |
| `unsafeQuery(sql, binds)` | `List<Map<String, Object>>` — no safety |

## Write execution

| Method | Returns |
|--------|---------|
| `insert(Map)` | `Long` generated id |
| `insertWithoutId(Map)` | `int` rows affected |
| `insertBatch(List<Map>)` | `int[]` batch results |
| `update(Map)` | `Integer` rows affected |
| `delete()` | `Integer` rows affected |
| `increment(col, long)` | `Integer` rows affected |
| `decrement(col, long)` | `Integer` rows affected |

## JoinOnBuilder (inside join lambda)

| Method | Description |
|--------|-------------|
| `where(col, val)` / `where(col, op, val)` | ON condition |
| `orWhere(...)` | OR condition |
| `whereIn(col, Object[])` / `orWhereIn(...)` | IN on ON |
| `whereNull(col)` / `whereNotNull(col)` | NULL on ON |
| `orWhereNull(col)` / `orWhereNotNull(col)` | OR NULL on ON |
| `colEq(left, right)` | Column = column |
| `colEq(left, right, joiner)` | Column = column with AND/OR |
| `groupStart()` / `orGroupStart()` / `groupEnd()` | Grouping |

## Page&lt;T&gt; record

| Method | Returns |
|--------|---------|
| `items()` | `List<T>` unmodifiable |
| `total()` | `long` |
| `page()` | `int` |
| `perPage()` | `int` |
| `totalPages()` | `int` |
| `hasNext()` | `boolean` |
| `hasPrevious()` | `boolean` |

## LikeMode enum

| Value | Pattern |
|-------|---------|
| `LEFT` | `%value` |
| `RIGHT` | `value%` |
| `BOTH` | `%value%` |

## Dialect instances

| Class | Constant |
|-------|----------|
| `GenericDialect` | `INSTANCE` |
| `PostgresDialect` | `INSTANCE` |
| `H2Dialect` | `INSTANCE` |
| `MysqlDialect` | `INSTANCE` |
| `MariaDbDialect` | `INSTANCE` |
| `SqlServerDialect` | `INSTANCE` |
| `OracleDialect` | `INSTANCE` |

## Static validators (DB / Identifiers)

| Method | Description |
|--------|-------------|
| `DB.isValidName(str)` | Valid identifier (generic) |
| `DB.isValidName(str, allowAlias)` | With alias support |
| `DB.isValidAliasName(str)` | Valid alias |
| `Identifiers.isValid(str, dialect)` | Valid for dialect |
| `Identifiers.isPlainName(str)` | Bind key validation |

## Exceptions

| Class | Extends |
|-------|---------|
| `AppException` | `RuntimeException` |
| `InvalidIdentifierException` | `AppException` |
| `InvalidArgumentException` | `AppException` |
| `InvalidBindException` | `AppException` |
| `InvalidStateException` | `AppException` |
| `UnsafeOperationException` | `AppException` |

## Spring Boot properties

```yaml
binwi:
  dataflow:
    dialect: generic | postgres | mysql | mariadb | h2 | sql_server | oracle
```

## Allowed WHERE operators

`=`, `!=`, `<>`, `>`, `<`, `>=`, `<=`, `like`, `ilike`, `in`, `not in`, `between`, `is null`, `is not null`

Note: use dedicated methods for `in`, `not in`, and `between`.

## Further reading

- [Querying Data](querying-data.md) — detailed WHERE and SELECT usage
- [Writing Data](writing-data.md) — INSERT/UPDATE/DELETE
- [Safety & Exceptions](safety-and-exceptions.md) — error handling
