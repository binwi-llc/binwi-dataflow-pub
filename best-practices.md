# Best Practices

- [Introduction](#introduction)
- [Thread Safety](#thread-safety)
- [Transactions](#transactions)
- [Prefer the Fluent API](#prefer-the-fluent-api)
- [Keep User Input in Binds](#keep-user-input-in-binds)
- [Use Typed Results for Known Shapes](#use-typed-results-for-known-shapes)
- [Close Your Streams](#close-your-streams)
- [Preview SQL in Tests](#preview-sql-in-tests)
- [Choose the Right Read Method](#choose-the-right-read-method)

<a name="introduction"></a>
## Introduction

DataFlow is small and predictable, but a handful of habits will keep your usage clean, safe,
and performant. This guide collects the patterns worth adopting from day one.

<a name="thread-safety"></a>
## Thread Safety

`db.table(...)` returns a **fresh** builder instance on every call, sharing only the
immutable `NamedParameterJdbcTemplate` and `Dialect`. A single `DB` instance is therefore
safe to use as a stateless factory across threads — every chain starts with `table(...)`
and operates on its own instance.

The instance returned by `table(...)` is itself mutable and is being assembled as you chain
methods, so **do not share a single in-progress chain across threads**. Build each query on
the thread that runs it.

Under Spring Boot, the auto-configured `DB` bean is prototype-scoped, giving each injection
point its own instance as an extra layer of isolation. If you wire `DB` manually as a
singleton, the factory guarantee above still makes it safe to share.

<a name="transactions"></a>
## Transactions

DataFlow has no transaction helper of its own — by design. Use Spring's transaction
management around your `DB` calls exactly as you would with any Spring JDBC code:

```java
@Service
public class TransferService {

    private final DB db;

    public TransferService(DB db) {
        this.db = db;
    }

    @Transactional
    public void transfer(long from, long to, long amount) {
        db.table("accounts").where("id", from).decrement("balance", amount);
        db.table("accounts").where("id", to).increment("balance", amount);
    }
}
```

Both statements run in the surrounding transaction and commit or roll back together.

<a name="prefer-the-fluent-api"></a>
## Prefer the Fluent API

The fluent methods carry every safety guarantee — identifier validation, parameter binding,
destructive-operation refusal. Reach for the raw escape hatches only when the fluent API
genuinely can't express what you need, and reserve `unsafeQuery` for the rare cases nothing
else fits. See [Raw SQL Escape Hatches](raw-sql-escape-hatches.md).

<a name="keep-user-input-in-binds"></a>
## Keep User Input in Binds

Never build a SQL fragment by concatenating user input, even into a raw helper. The token
filter (`;`, `--`, `/*`, `*/`) is a backstop, not a sanitizer. Put values in the bind map:

```java
// Good — the value is bound
db.table("users").whereRaw("last_login > :cutoff", Map.of("cutoff", userCutoff));

// Never — concatenating user input into the fragment
db.table("users").whereRaw("last_login > '" + userCutoff + "'");
```

Identifiers that come from user input should be validated with `DB.isValidName(...)` before
use, or better, mapped through an allow-list you control.

<a name="use-typed-results-for-known-shapes"></a>
## Use Typed Results for Known Shapes

When a query's result shape is known and stable, map it into a record with `get(Class)`,
`first(Class)`, or `paginate(..., Class)`. You get immutable, well-named objects and
automatic `snake_case` → `camelCase` mapping. Keep the `Map<String, Object>` form for
genuinely dynamic queries. See [Typed Results](typed-results.md).

<a name="close-your-streams"></a>
## Close Your Streams

`stream()`, `stream(Class)`, and `stream(RowMapper)` are backed by an open JDBC cursor.
Always consume them inside try-with-resources so the cursor — and its connection — is
released:

```java
try (Stream<User> users = db.table("users").orderByAsc("id").stream(User.class)) {
    users.forEach(this::process);
}
```

A leaked stream leaks a connection. See
[Pagination & Streaming](pagination-and-streaming.md#streaming-large-result-sets).

<a name="preview-sql-in-tests"></a>
## Preview SQL in Tests

`toSql()` renders a read query's SQL and bind map without executing it. Use it to assert the
exact statement in unit tests — no database required:

```java
SqlAndParams preview = db.table("users").where("active", true).take(10).toSql();

assertThat(preview.sql()).contains("WHERE active = :active_1");
assertThat(preview.params()).containsEntry("active_1", true);
```

<a name="choose-the-right-read-method"></a>
## Choose the Right Read Method

Match the terminal method to your access pattern:

| Method | Use when |
|--------|----------|
| `get()` / `get(Class)` | You need all matching rows and they fit comfortably in memory. |
| `first()` / `first(Class)` | You expect at most one row. |
| `paginate(...)` | You're rendering page controls with totals. |
| `stream(...)` | You're processing a very large set once, top to bottom. |
| `count()` | You only need the number of matching rows. |

Picking the right one keeps memory bounded and avoids unnecessary work.
</content>
