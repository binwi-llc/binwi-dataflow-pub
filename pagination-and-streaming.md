# Pagination & Streaming

- [Introduction](#introduction)
- [Page-Based Pagination](#page-based-pagination)
- [The Page Object](#the-page-object)
- [Typed Pagination](#typed-pagination)
- [Manual Limit and Offset](#manual-limit-and-offset)
- [Streaming Large Result Sets](#streaming-large-result-sets)
- [Choosing Between Them](#choosing-between-them)

<a name="introduction"></a>
## Introduction

DataFlow offers two strategies for working with result sets that don't fit a single flat
`List`: **pagination**, for page-by-page navigation with totals, and **streaming**, for
iterating very large sets lazily through a database cursor. This guide covers both.

<a name="page-based-pagination"></a>
## Page-Based Pagination

`paginate(perPage, page)` runs two queries — a `COUNT` for the total and a windowed
`SELECT` for the page — and returns a typed `Page`:

```java
Page<Map<String, Object>> page = db.table("users")
        .where("active", true)
        .orderByDesc("created_at")
        .paginate(20, 1);
```

Both arguments must be greater than zero, or an `InvalidArgumentException` is thrown. Pages
are 1-indexed.

> [!WARNING]
> On SQL Server, always include an `orderByAsc`/`orderByDesc` before paginating — its
> `OFFSET ... FETCH` syntax requires an ordering. See
> [Dialects & Identifiers](dialects-and-identifiers.md#pagination-by-dialect).

<a name="the-page-object"></a>
## The Page Object

`Page<T>` is an immutable `record` in `com.binwi.dataflow.Page`. It carries the items plus
everything you need to render pagination controls:

```java
page.items();        // List<T> for this page (defensively copied, unmodifiable)
page.total();        // long — total rows ignoring LIMIT/OFFSET
page.page();         // 1  — current page (1-indexed)
page.perPage();      // 20 — page size
page.totalPages();   // computed from total / perPage
page.hasNext();      // true if a following page exists
page.hasPrevious();  // true if a preceding page exists
```

`Page<T>` also implements `Iterable<T>`, so you can loop over it directly:

```java
for (Map<String, Object> row : page) {
    // ...
}
```

<a name="typed-pagination"></a>
## Typed Pagination

Pass a class or a `RowMapper` as the third argument to paginate straight into typed objects:

```java
Page<User> page = db.table("users")
        .orderByDesc("created_at")
        .paginate(20, 1, User.class);
```

The mapping rules are identical to `get(Class)` — see [Typed Results](typed-results.md).

<a name="manual-limit-and-offset"></a>
## Manual Limit and Offset

When you don't need totals — an infinite-scroll feed, say — use `take` and `skip` directly
and call `get()`:

```java
List<Map<String, Object>> feed = db.table("posts")
        .orderByDesc("created_at")
        .take(20)
        .skip(40)
        .get();
```

Both values are bound as parameters (`:_limit`, `:_offset`) and must be non-negative.

<a name="streaming-large-result-sets"></a>
## Streaming Large Result Sets

For result sets too large to hold in memory — millions of rows — use `stream()`. It returns
a `java.util.stream.Stream` backed by a live JDBC cursor, so rows are fetched lazily as you
consume them:

```java
try (Stream<User> users = db.table("users")
        .where("active", true)
        .orderByAsc("id")
        .stream(User.class)) {
    users.forEach(this::process);
}
```

> [!WARNING]
> The stream is backed by an open cursor and **must be closed**. Always use
> try-with-resources. Leaking a stream leaks a database connection.

Three overloads are available, mirroring the read API:

- `stream()` — rows as `Map<String, Object>`.
- `stream(Class<T>)` — rows mapped to a record/POJO.
- `stream(RowMapper<T>)` — rows mapped by your own `RowMapper`.

<a name="choosing-between-them"></a>
## Choosing Between Them

| Use | When |
|-----|------|
| `paginate(...)` | Rendering page controls, showing "page X of Y", jumping to a page. |
| `take` / `skip` + `get()` | Simple windowing without needing a total count. |
| `stream(...)` | Processing a very large set once, top to bottom, without materializing it. |

Pagination runs an extra `COUNT` query; streaming holds a cursor open. Pick the one that
matches your access pattern.
</content>
