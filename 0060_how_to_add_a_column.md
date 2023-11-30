Originally from: [tweet](https://twitter.com/samokhvalov/status/1728440121899487649), [LinkedIn post]().

---

# How to add a column

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Adding a column is straightforward:

```sql
alter table t1 add column c1 int8;

comment on column t1.c1 is 'My column';
```

However, there are a few potential complexities.

## Locking issues

Adding a column requires table-level `AccessExclusiveLock`, which blocks all queries to the table including `SELECT`s. 
Two consequences of it:

1) We don't want the operation to last long (e.g. scanning the whole table or, even worse, rewriting it).
2) The lock acquisition should be done gracefully.

Regarding the latter, it's analyzed in detail in 
[Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries).

An example of graceful approach, with low `lock_timeout` and retries:

```sql
do $do$
declare
   lock_timeout constant text := '50ms';
   max_attempts constant int := 1000;
   ddl_completed boolean := false;
begin

   perform set_config('lock_timeout', lock_timeout, false);

   for i in 1..max_attempts loop
      begin
         execute 'alter table t1 add column c1 int8';
         ddl_completed := true;
         exit;
      exception
         when lock_not_available then
           null;
      end;
   end loop;

   if ddl_completed then
      raise info 'DDL successfully executed';
   else
      raise exception 'DDL execution failed';
   end if;
end $do$;
```

Note that in this particular example, subtransactions are implicitly used (the `BEGIN/EXCEPTION WHEN/END` block). Which
can be a problem in case of very high XID growth rate (e.g., many writing transactions) and a long-running transaction –
this can trigger SubtransSLRU contention on standbys; see: 
[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful). 
In this case, implement the retry logic at transaction level.

## DEFAULT

Since Postgres 11 (so, in all currently supported major versions, as of November 2023), adding a column with a default
value does not lead to full table rewrite, so we generally do not need to worry about it.

The DEFAULT value defined at column creation is not written to all existing rows physically; instead, it's "virtual" –
stored in the special system catalog [pg_attrdef](https://postgresql.org/docs/current/catalog-pg-attrdef.html).
Demonstration:

```sql
nik=# alter table t1 add column c2 int8 default -10;
ALTER TABLE

nik=# select pg_get_expr(adbin, 't1'::regclass::oid) from pg_attrdef;
  pg_get_expr
----------------
 '-10'::integer
(1 row)
```

While the values for `DEFAULT` defined for already existing columns are stored together with column definition, in 
`pg_attribute`:

```sql
nik=# alter table t1 alter column c2 set default -20;
ALTER TABLE

nik=# select attmissingval from pg_attribute where attrelid = 't1'::regclass::oid and attname = 'c2';
 attmissingval
---------------
 {-10}
(1 row)
```

This also means that, when adding a new column, if we want to "virtually" backfill all existing rows with one value but
use another default value for all future rows, we can:

- use one `DEFAULT` value at column creation time,
- change `DEFAULT` to a different value right after creation.

If you use very old Postgres version (pre-11), consider to use backfilling to avoid long-lasting locking.

## NOT NULL

Adding a NOT NULL constraint (that is required for a PK [re]definition), generally, requires a full-table scan there is
no support of two-step addition to avoid long-lasting locking. However, when this constraint is needed for a new column,
we can use this trick

1) Use some temporary DEFAULT combined with NOT NULL at column creation:

   ```sql
   alter table t1
   add column id_new int8 not null default -1;
 
   comment on column "t1"."id_new" is 'my future PK'; 
   ```

2) Take care of the existing and new-coming values in this column:

- install a trigger to auto-fill values in new rows,
- backfill the values in existing rows, in batches.

3) Once backfilling is finished, remove the temporary `DEFAULT`:

   ```sql
   alter table t1 alter column id_new drop default;
   ```

4) Proceed with switching to the general use of this new column and then, if needed, cleanup (drop trigger, etc.)

## Backfilling

In some cases, the single-value `DEFAULT` is not enough to define the values in the new column for all existing rows, 
and we still need to backfill. This has to be done in batches, to avoid long-lasting locks. Notes:

1. As usual, for OLTP (web and mobile apps), it is recommended to find batch size so all `UPDATE`s do not exceed 1-2
   seconds.
2. To be able to efficiently find the scope for the next batch, we can create an index on the new column and existing
   PK (this index may be temporarily, to support efficient batching), and then drop at. This index can be partial. For
   example, if our new column is called `id_new` and the `DEFAULT` used at column creation time was `-1`:

- Create supporting index:

   ```sql
   create index concurrently i_t1_id_new on t1(id) where "id_new" = -1;
   ```
  
- For batching, to define scope for the UPDATE, use:

   ```sql
   ...
   where id_new = -1
   order by id
   limit :batch_size;
   ```

- Control the dead tuple counts and autovacuum behavior not to allow dead tuple count to be too high (leading to
  bloat) – throttle the frequency `UPDATE`s if needed and/or issue manual `VACUUM` from time to time.
- If the supporting index is not needed, drop it:

   ```sql
   drop index concurrently i_t1_id_new;
   ```

-----

## Correction regarding the internals of how DEFAULT values are stored

- `pg_attrdef` stores all current defaults. When we change `DEFAULT` for an existing column, this catalog is updated to
  store the new value:

   ```sql
   nik=# alter table t1 alter column c2 set default -30;
   ALTER TABLE
  
   nik=# select pg_get_expr(adbin, 't1'::regclass::oid) from pg_attrdef;
     pg_get_expr
   ----------------
    '-30'::integer
   (1 row)
   ```

   And the value stored in `pg_attribute` in `attmissingval` is that one that is used for the rows that existed before
   column was created:

   ```sql
   nik=# select attmissingval from pg_attribute where attrelid = 't1'::regclass::oid and attname = 'c2';
    attmissingval
   ---------------
    {-10}
   (1 row)
   ```
