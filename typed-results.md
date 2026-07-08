# Typed Results

- [Introduction](#introduction)
- [Mapping to Records](#mapping-to-records)
- [Mapping to POJOs](#mapping-to-pojos)
- [snake_case to camelCase](#snake_case-to-camelcase)
- [First and Paginate](#first-and-paginate)
- [Custom RowMappers](#custom-rowmappers)
- [When to Stay Untyped](#when-to-stay-untyped)

<a name="introduction"></a>
## Introduction

By default, DataFlow returns rows as `Map<String, Object>`. That's convenient for ad-hoc
work, but most of the time you want typed objects. Pass a `Class` to `get`, `first`, or
`paginate`, and DataFlow maps each row into an instance using Spring's
`DataClassRowMapper` — which understands both Java records and classic beans.

<a name="mapping-to-records"></a>
## Mapping to Records

A Java record is the most concise target. Define one whose components match your columns:

```java
public record User(Long id, String name, String email, Boolean active,
                   Integer age, String role, Integer score,
                   Timestamp createdAt, Timestamp updatedAt) {}
```

Then map straight into it:

```java
List<User> users = db.table("users").orderByAsc("id").get(User.class);
```

Each row's columns are matched to the record's components by name, converting `snake_case`
column names to `camelCase` components automatically.

<a name="mapping-to-pojos"></a>
## Mapping to POJOs

Plain classes with a default constructor and setters (JavaBeans) work too:

```java
public class UserBean {
    private Long id;
    private String name;
    private String email;
    // getters and setters ...
}

List<UserBean> users = db.table("users").get(UserBean.class);
```

Records are generally preferable for read models — they're immutable and require no
boilerplate — but the choice is yours.

<a name="snake_case-to-camelcase"></a>
## snake_case to camelCase

`DataClassRowMapper` bridges the naming conventions of the two worlds. A column named
`created_at` maps to a `createdAt` component or property; `user_id` maps to `userId`. You
don't need annotations or configuration for this — it is the default behavior.

This means your SQL can follow database conventions (`snake_case`) while your Java follows
Java conventions (`camelCase`), with no manual mapping in between.

<a name="first-and-paginate"></a>
## First and Paginate

The typed overloads exist for the single-row and paginated terminals too:

```java
Optional<User> user = db.table("users").where("id", 42L).first(User.class);

User u = db.table("users")
        .where("id", 42L)
        .first(User.class)
        .orElseThrow(() -> new NoSuchElementException("user not found"));

Page<User> page = db.table("users")
        .orderByDesc("created_at")
        .paginate(20, 1, User.class);
```

`stream(Class<T>)` is also available for lazy, typed iteration — see
[Pagination & Streaming](pagination-and-streaming.md).

<a name="custom-rowmappers"></a>
## Custom RowMappers

When you need control that name-based mapping can't give — projecting a single column,
combining fields, or handling an unusual type — pass a `RowMapper<T>` instead of a `Class`:

```java
List<String> emails = db.table("users")
        .select("email")
        .get((rs, rowNum) -> rs.getString("email"));
```

The `RowMapper` overload is available on `get`, `first`, `paginate`, and `stream`, so you
can use custom mapping in every read path.

<a name="when-to-stay-untyped"></a>
## When to Stay Untyped

The `Map<String, Object>` form is still the right choice for genuinely dynamic queries —
reporting tools, admin consoles, or anywhere the column set isn't known at compile time:

```java
List<Map<String, Object>> rows = db.table(userProvidedTable).get();
```

Reach for typed results when the shape is known and stable; stay with maps when it isn't.
</content>
