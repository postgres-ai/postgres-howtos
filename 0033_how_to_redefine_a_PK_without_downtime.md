Originally from: [tweet](https://twitter.com/samokhvalov/status/1718513041829187856), [LinkedIn post]().

---

# How to redefine a PK without downtime

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Redefining a PK (primary key) is not a difficult procedure, yet it requires a few non-trivial steps to be done. This
procedure also is a part of the `int4`->`int8` PK conversion when using a "new column" method (to be discussed
separately).

Of course, we can just drop a PK and define a new one like this:

```sql
alter table your_table
drop constraint your_table_pkey;

alter table your_table
add primary key (new_column);
```

But this straightforward approach is, in general, a terrible idea, because it will acquire an `AccessExclusiveLock` on
the table and hold it for a long time because:

1. it needs to build a `UNIQUE` constraint,
2. it needs to build a `NOT NULL` constraint.

This is because to build a PK, we need two ingredients: a `UNIQUE` constraint, and `NOT NULL` on all columns
participating in the PK definition. Fortunately, in modern Postgres (PG12+), it is possible to avoid long-lasting
exclusive locks – in other words, to have truly "online" or "zero-downtime" operation.

Below we assume that:

- the new PK's column(s) is (are) already exists and pre-filled
- and continue to be filled if more `INSERT`s and `UPDATE`s are coming – so the data is already there,
- the uniqueness property is achieved for the column(s), and
- no `NULL` values are present in these column(s).

Note, that the last condition is essential – unlike UKs (unique key), a PK requires all columns participating in its
definition to have a `NOT NULL` constraint.

## NOT NULL: many good and bad news (eventually, all good)

Let's dive into details here – NOT NULL deserves it. We'll have a bunch of good and bad news. We'll dive into specifics
that are not necessarily related to PK, but they are still relevant. And eventually we'll return to the PK redefinition
task. Just bear with me.

**Bad news:** unfortunately, adding a NOT NULL constraint to an existing column means that Postgres will need to perform a
long (for large tables) full-table scan, during which it will an `AccessExclusiveLock` acquired by `ALTER TABLE` is
going to be held. This is not what we want if we need zero-downtime operations.

**Good news:** since Postgres 11, we can execute a trick, if we need to add a column with `NOT NULL` – we can benefit from
PG11's new feature, non-blocking `DEFAULT` for new columns, and we combine it with `NOT NULL`, for example:

```sql
alter table t1
add column new_id int8 not null default -1;
```

This is very fast, thanks to PG11's optimization of `DEFAULT` for new columns (it's "virtual" – no whole-table rewrite
happens):

> ability to avoid a table rewrite for `ALTER TABLE ... ADD COLUMN` with a non-null column default
>
> ([PG11 release notes](https://postgresql.org/docs/release/11.0/))

And since all rows are pre-filled ("virtually", but it doesn't matter), we can have `NOT NULL` right away, avoiding long
wait.

**Bad news:** this works only for new columns. If we deal with an existing column, and still want to add a `NOT NULL` to it,
this won't work.

**Good news:** if we just need a "not null", not matter how defined, we can use a `CHECK` constraint. The good thing about
`CHECK` constraints is that their definition can be two-phase:

- first, we define a constraint `CHECK (col1 IS NOT NULL)` with flag `NOT VALID` – this is fast, not blocking other
  sessions because the existing rows are not checked (well, blocking – it's still an `ALTER TABLE` – but for a very
  brief moment of time; of course, retries and low `lock_timeout` are still needed,
  see [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)),
- second, we perform the "validation", using `ALTER TABLE ... VALIDATE CONSTRAINT ...` – this is slow, but fortunately,
  not blocking others.

**Bad news:** since our final goal is a PK redefinition, the `CHECK` constraint won't work for us because PK requires a
truly `NOT NULL` one.

**Good news:** in PG12+, there is an optimization that allows `NOT NULL` constraint definition to rely on an existing
`CHECK (... IS NOT NULL)` constraint:

> Allow ALTER TABLE ... SET NOT NULL to avoid unnecessary table scans
>
> ([PG12 release notes](https://postgresql.org/docs/release/12.0/))

This means that we just need to do this:

1. Create a `CHECK` constraint ensuring IS NOT NULL for our column(s), marked as `NOT VALID` (acquiring a brief
   exclusive lock with low `lock_timeout` and, if needed, multiple retries)
2. In a separate transaction, `VALIDATE` it
3. Then, add `NOT NULL` constraint on the column(s) – it is going to be fast now! (again, low `lock_timeout` and
   retries).
4. drop the `CHECK` constraint (again, low `lock_timeout` and retries).

Interestingly, it is okay to skip step 3 here if our final goal is a PK creation – the `NOT NULL` constraint will be
created implicitly, during PK creation; and it will be fast thanks to already existing `CHECK (... NOT NULL)`.

## UNIQUE

The second ingredient we need for a new PK creation is a `UNIQUE` constraint. Fortunately, it can be created in two
phases, avoiding long-lasting exclusive locks:

1. Create a unique index, in "zero-downtime" fashion, thanks to the `CONCURRENTLY` option – and it is important to give
   this index a name, because we'll use this name later:

```sql
create unique index concurrently new_unique_index
on your_table using btree(your_column);
```

2. Use this index when defining a PK (... `USING INDEX` ...)

## The whole recipe

Now, let's complete the puzzle and see the whole picture.

Building a new PK in zero-downtime fashion consists of these five steps:

1. Create a `CHECK (... IS NOT NULL)` constraint with the `NOT VALID` option ✂️:

```sql
alter table your_table
add constraint your_table_your_column_check
  check (your_column is not null) not valid;
```

2. `VALIDATE` the constraint (takes time):

```sql
alter table your_table
validate constraint your_table_your_column_check;
```

3. Build a unique index, using the `CONCURRENTLY` option:

```sql
create unique index concurrently u_your_table_your_column
on your_table using btree(your_column);
```

4. Define a PK based on the existing unique index and `CHECK` constraint (implicitly creating `NOT NULL` skipping full
   table scan)✂️:

```sql
alter table your_table
add constraint your_table_pkey primary key
using index u_your_table_your_column;
```

5. Drop the `CHECK` constraint to clean up✂️:

```sql
alter table your_table
drop constraint your_table_your_column_check;
```

(✂️ – recommended to use low `lock_timeout` and retries)
