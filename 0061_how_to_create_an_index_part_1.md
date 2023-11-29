Originally from: [tweet](https://twitter.com/samokhvalov/status/1728800785889395192), [LinkedIn post]().

---

# How to create an index, part 1

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Index creation is straightforward:

```sql
create index concurrently on t1(c1);
```

A longer form with explicit index naming:

```sql
create index concurrently i_1 on t1(c1);
```

And even longer, explicitly including both index name and type:

```sql
create index concurrently i_1
on t1
using btree(c1);
```

Below are a few "best practices" considerations.

## Index names

When creating indexes:

1. Use explicit naming for better control.
2. Establish and follow some naming schema. For example, including the names of the table and columns to the index
   name: `i_table1_col1_col2`. The other properties to consider for inclusion:
    - is it a regular index or unique?
    - index type
    - is it a partial index?
    - ordering, expressions used
    - opclasses used

## Settings

1) `statement_timeout`: If you have `statement_timeout` set globally, unset it in the session where you are building an
   index:

   ```sql
   set statement_timeout to 0;
   ```

   Alternatively, you can create a special DB user for index creation and adjust `statement_timeout` for it:

   ```sql
   alter user index_creator set statement_timeout to 0;
   ```

2) `maintenance_work_mem`: raise this parameter ([docs](https://postgresqlco.nf/doc/en/param/maintenance_work_mem/)) to
   hundreds of MiB or a few GiB (on larger systems) to support faster index creation.

## CONCURRENTLY

Always prefer using the option `CONCURRENTLY` unless:

- you're building an index on a table that is known not to be used yet – e.g., a table that was just created (it is
  beneficial to avoid `CONCURRENTLY` in this case, to be able to include `CREATE INDEX` in a transaction block together
  with index creation);
- you're working with database alone.

`CONCURRENTLY` will increase index build time, but will handle locking gracefully, not blocking other sessions for long
time. With this method, an index is built with a careful balance between allowing ongoing access to the table while
creating a new index, maintaining data integrity and consistency, and minimizing disruptions to normal database
operations.

👉 How it is
[implemented in PG16](https://github.com/postgres/postgres/blob/c136eb02981566d56e950f12ab7ee4a6ea51d698/src/backend/catalog/index.c#L1443-L1511).

When this option is used, index creation might fail due to various reasons – for example, if you make attempts to build
two indexes in parallel, for one of the attempts you'll see something like this:

```
nik=# create index concurrently i_3 on t1 using btree(c1);
ERROR:  deadlock detected
DETAIL:  Process 518 waits for ShareLock on virtual transaction 3/459; blocked by process 553.
Process 553 waits for ShareUpdateExclusiveLock on relation 16402 of database 16401; blocked by process 518.
HINT:  See server log for query details.
```

In general, Postgres has transaction support for DDL, but for `CREATE INDEX CONCURRENTLY`, it is not so:

- you cannot include `CREATE INDEX CONCURRENTLY` to a transaction block,
- if operation fails, it leaves an invalid index behind, so a cleanup is needed.

## Cleanup and retries

Since we know that `CREATE INDEX CONCURRENTLY` might fail, we should be ready to retry, manually or automatically. 
Before retrying, we need to cleanup an invalid index.
Here the use of explicit naming and some schema convention pays off.

When cleaning up, also use `CONCURRENTLY`:

```sql
drop index concurrently i_1;
```

## Progress monitoring

How to monitor index creation progress: See
[Day 15: How to monitor CREATE INDEX / REINDEX progress in Postgres 12+](0015_how_to_monitor_index_operations.md).

## ANALYZE

Generally, Postgres `autovacuum` maintains statistics for each column up-to-date, running `ANALYZE` for each table whose
content changes enough.

After a new index creation, usually there is no need to rebuild statistics if you index columns only.

However, if you: build an index on an expression, e.g.:

```sql
create index i_t1_lower_email on t1 (lower(email));
```

Then you should run `ANALYZE` on the table so Postgres gathers statistics for expression:

```sql
analyze verbose t1;
```
