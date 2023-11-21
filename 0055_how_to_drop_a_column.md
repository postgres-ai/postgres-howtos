Originally from: [tweet](https://twitter.com/samokhvalov/status/1726596564712571006), [LinkedIn post]().

---

# How to drop a column

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Dropping a column is easy:

```sql
alter table t1 drop column c1;
```

However, it is important to keep in mind a few complications that might occur in various situations.

## Risk 1: application code not ready

Application code needs to stop using this column. It means that it needs to be deployed first.

## Risk 2: partial downtime

Under heavy load, issuing such an alter without a low `lock_timeout` and retries is a bad idea because this statement
need to acquire AccessExclusiveLock on the table, and if an attempt to acquire it lasts a significant time (e.g. because
of existing transaction that holds any lock on this table - it can be a transaction that read a single row from this
table, or autovacuum processing this table to prevent transaction ID wraparound), then this attempt can be harmful for
all current queries to this table, since it will be blocking them. This causes partial downtime in projects under load.
Solution: low `lock_timeout` and retries. An example (more about this and a more advanced example can be found
in [zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)):

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
         execute 'alter table t1 drop column c1';
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
can be a problem in case of very high `XID` growth rate (e.g., many writing transactions) and a long-running
transaction – this can trigger `SubtransSLRU` contention on standbys (see:
[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful)).
In this case, implement the retry logic at transaction level.

## Risk 3: false expectations that the data is deleted

Finally, when copying data between various environments and removing sensitive data, remember that
`ALTER TABLE ... DROP COLUMN ...` is not secure, it doesn't remove the data. After column `c1` is dropped, there is
still information about it in metadata:

```sql
nik=# select attname from pg_attribute where attrelid = 't1'::regclass::oid order by attnum;
           attname
------------------------------
 tableoid
 cmax
 xmax
 cmin
 xmin
 ctid
 id
 ........pg.dropped.2........
(8 rows)
```

A superuser can easily recover it:

```sql
nik=# update pg_attribute
  set attname = 'c1', atttypid = 20, attisdropped = false
  where attname = '........pg.dropped.2........';
UPDATE 1
nik=# \d t1
                 Table "public.t1"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 id     | bigint  |           |          |
 c1     | bigint  |           |          |
```

Some solutions to this problem:

- `VACUUM FULL` to rebuild the table after dropping columns. In this case, though a restoration attempt will succeed,
  the data will not be there.
- Consider using restricted users and column-level privileges instead of dropping columns. The columns and data will
  remain, but users will not be able to read it. Of course, this approach wouldn't suit if there is a strict requirement
  to remove data.
- Dump/restore after the column has already been dropped.
