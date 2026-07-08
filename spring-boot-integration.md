# Spring Boot Integration

- [Introduction](#introduction)
- [How Auto-Configuration Works](#how-auto-configuration-works)
- [Configuring the Dialect](#configuring-the-dialect)
- [Bean Scope](#bean-scope)
- [Providing Your Own DB Bean](#providing-your-own-db-bean)
- [Providing Your Own Dialect Bean](#providing-your-own-dialect-bean)
- [Multiple Datasources](#multiple-datasources)

<a name="introduction"></a>
## Introduction

When DataFlow is on the classpath of a Spring Boot application, it configures itself. Drop
in the dependency, make sure a `NamedParameterJdbcTemplate` exists, and inject `DB`
anywhere you need it. This guide explains exactly what the auto-configuration does and how
to override it when you need finer control.

<a name="how-auto-configuration-works"></a>
## How Auto-Configuration Works

The auto-configuration activates when **all** of the following are true:

1. `NamedParameterJdbcTemplate` is on the classpath.
2. A bean of that type exists in the application context.
3. No user-defined `DB` bean already exists.

When these conditions hold, DataFlow registers two beans:

- A `Dialect` bean, driven by the `binwi.dataflow.dialect` property (defaulting to `generic`).
- A `DB` bean that wraps the context's `NamedParameterJdbcTemplate` with that dialect.

If any condition fails — for example, you declare your own `DB` — the auto-configuration
backs off and leaves your definition untouched.

<a name="configuring-the-dialect"></a>
## Configuring the Dialect

Select your database dialect with a single property. In `application.yml`:

```yaml
binwi:
  dataflow:
    dialect: postgres   # generic | postgres | mysql | mariadb | h2 | sql_server | oracle
```

Or in `application.properties`:

```properties
binwi.dataflow.dialect=postgres
```

The value maps to the built-in dialects. If you omit the property, DataFlow uses
`generic`, which is backward compatible with pre-dialect code. See
[Dialects & Identifiers](dialects-and-identifiers.md) for what each dialect changes.

<a name="bean-scope"></a>
## Bean Scope

The auto-configured `DB` bean is registered with **prototype** scope
(`@Scope("prototype")`). Each injection point receives its own instance.

In practice this rarely matters, because `db.table(...)` already returns a fresh builder on
every call — the `DB` you hold is effectively a stateless factory. The prototype scope is
simply an extra layer of isolation. See [Best Practices](best-practices.md#thread-safety)
for the full thread-safety story.

<a name="providing-your-own-db-bean"></a>
## Providing Your Own DB Bean

To take full control of construction, declare a `DB` bean yourself. The auto-configuration
detects it and steps aside:

```java
@Configuration
public class DataflowConfig {

    @Bean
    public DB db(NamedParameterJdbcTemplate jdbc) {
        return new DB(jdbc, PostgresDialect.INSTANCE);
    }
}
```

<a name="providing-your-own-dialect-bean"></a>
## Providing Your Own Dialect Bean

If you only want to change the dialect — not the whole `DB` — declare a `Dialect` bean. The
auto-configured `DB` will pick it up:

```java
@Configuration
public class DataflowConfig {

    @Bean
    public Dialect dialect() {
        return PostgresDialect.INSTANCE;
    }
}
```

This is handy when you want a custom `Dialect` subclass, or when you would rather express
the choice in Java than in a property.

<a name="multiple-datasources"></a>
## Multiple Datasources

For applications with more than one datasource, wire a dedicated `DB` per template. Use
`@Qualifier` to select the right `NamedParameterJdbcTemplate`:

```java
@Configuration
public class DataflowConfig {

    @Bean
    public DB analyticsDb(@Qualifier("analyticsJdbc") NamedParameterJdbcTemplate jdbc) {
        return new DB(jdbc, PostgresDialect.INSTANCE);
    }

    @Bean
    public DB reportingDb(@Qualifier("reportingJdbc") NamedParameterJdbcTemplate jdbc) {
        return new DB(jdbc, MysqlDialect.INSTANCE);
    }
}
```

Because you are declaring `DB` beans explicitly, the auto-configuration backs off — you own
the wiring completely, including which dialect each datasource uses.
</content>
