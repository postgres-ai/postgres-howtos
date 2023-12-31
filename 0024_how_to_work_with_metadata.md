Originally from: [tweet](https://twitter.com/samokhvalov/status/1715249090202870113), [LinkedIn post]().

---

# How to work with metadata

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

When working with metadata – data about data – in Postgres, these reference docs are worth using:

- [System Catalogs](https://postgresql.org/docs/current/catalogs.html)
- [System Views](https://postgresql.org/docs/current/views.html)
- [The Cumulative Statistics System](https://postgresql.org/docs/current/monitoring-stats.html)
- [The Information Schema](https://postgresql.org/docs/current/information-schema.html)

It's unnecessary to repeat the material from the docs here. Instead, let's focus on some tricks and principles that make
your work more efficient. We'll cover these topics:

- `::oid`, `::regclass`
- `\?` and `ECHO_HIDDEN`
- Performance
- `INFORMATION_SCHEMA`
- `pg_stat_activity` is not a table

## ::oid, ::regclass

In Postgres terminology, tables, indexes, views, materialized views are all called "relations". The metadata about them
can be seen in various ways, but the "central" place is the
[pg_class system catalog](https://postgresql.org/docs/current/catalog-pg-class.html). In other words, this is a tables
that stores
information about all tables, indexes, and so on. It has two keys:

- PK: `oid` - a number ([OID, object identifier](https://postgresql.org/docs/current/datatype-oid.html))
- UK: a pair of columns `(relname, relnamespace)`, relation name and OID of the schema.

A trick to remember: OID can be quickly converted to relation name, vice versa, using type conversion to `oid` and
`regclass` datatypes.

Simple examples for a table named `t1`:

```
nik=# select 't1'::regclass;
 regclass
----------
 t1
(1 row)

nik=# select 't1'::regclass::oid;
  oid
-------
 74298
(1 row)

nik=# select 74298::regclass;
 regclass
----------
 t1
(1 row)
```

So, there is no need to do `select oid from pg_class where relname = ...` – just memorize `::regclass` and `::oid`.

## \? and ECHO_HIDDEN

`psql`'s `\?` command is crucial – this is how you can find description for all commands. For example:

```
\d[S+]                 list tables, views, and sequences
```

The "describing" commands produce some SQL implicitly – and it can be helpful to "spy" on them. For that, we first need
to turn on `ECHO_HIDDEN`:

```
nik=# \set ECHO_HIDDEN on
```

– or just use the option `-E` when starting `psql`. And then we can start spying:

```
nik=# \d t1
/********* QUERY **********/
SELECT c.oid,
 n.nspname,
 c.relname
FROM pg_catalog.pg_class c
    LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
 AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 2, 3;
/**************************/

[... + more queries to get info about "t1" ...]
```

Examining these queries can assist in building various tooling to work with metadata.

## Performance

In some cases, metadata queries can be heavy, slow. Here's what to do if it's so:

1. Consider caching to reduce the frequency and necessity of metadata queries.

2. Check for catalog bloat. For example `pg_class` can be bloated due to frequent DDL, use of temp tables, etc. In this
   case, unfortunately, a `VACUUM FULL` is needed (`pg_repack` cannot repack system catalogs). If you need it, don't
   forget the golden rule of zero-downtime DDLs in Postgres –
   [use low lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries).

## INFORMATION_SCHEMA

System catalogs and views are "native" ways to query table and index metadata – not standard, though. The standard way
is called `INFORMATION_SCHEMA` and Postgres supports it following the SQL
standard: [Docs](https://postgresql.org/docs/current/information-schema.html). What to use:

- Use the information schema for simple, cross-database compatible metadata queries.
- Use native system catalogs for more complex, Postgres-specific queries or when you need detailed internal information.

## pg_stat_activity is not a table

It's essential to remember that when querying metadata, you might deal with something that doesn't behave as normal
table even if it looks so.

For instance, when you read records from `pg_stat_activity`, you're not dealing with a consistent snapshot of table
data: reading the first and, theoretically, the last rows are produced at different moments of time, and you might see
the queries which were not running simultaneously.

This phenomenon also explains why `select now() - query_start from pg_stat_activity;` might give you negative values:
the function `now()` is executed at the beginning of the transaction, once, and doesn't change its value inside the
transaction, no matter how many times you call it.

To get precise time intervals, use `clock_timestamp()` instead
(`select clock_timestamp() - query_start from pg_stat_activity;`).
