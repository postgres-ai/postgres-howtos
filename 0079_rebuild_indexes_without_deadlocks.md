Originally from: [tweet](https://twitter.com/samokhvalov/status/1735204785232715997), [LinkedIn post]().

---

# How to rebuild many indexes using many backends avoiding deadlocks

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Sometimes we need to reindex many indexes – or all of them – and want to do it faster.

For example, this makes sense after upgrading from pre-14 Postgres to version 14+, when we want to rebuild all B-tree
indexes to benefit from optimizations that reduce bloat growth rates.

We might decide to use a single session and higher value of
[max_parallel_maintenance_workers](https://postgresqlco.nf/doc/en/param/max_parallel_maintenance_workers/), processing
one index at a time. But if we have powerful resources (a lot of vCPUs and fast disks), then the maximal value of
`max_parallel_maintenance_workers` may not be enough to move as fast as we can (changing
`max_parallel_maintenance_workers` doesn't require a restart, but we cannot use more than `max_worker_processes`
workers, and changing that requires a restart). In this case, it may make sense to process multiple indexes in parallel,
using `REINDEX INDEX CONCURRENTLY`.

But in this case, indexes need to be processed in proper order. The problem is that if you attempt to rebuild two
indexes belonging to the same table in parallel, a deadlock will be detected, and one of the sessions will fail:

```sql
nik=# reindex index concurrently t1_hash_record_idx3;
ERROR:  deadlock detected
DETAIL:  Process 40 waits for ShareLock on virtual transaction 4/2506; blocked by process 1313.
Process 1313 waits for ShareUpdateExclusiveLock on relation 16634 of database 16401; blocked by process 40.
HINT:  See server log for query details.
```

To address this, we can use this approach:

1. Decide how many reindexing sessions you want to use, taking into account `max_parallel_maintenance_workers` and the
   planned resource utilization / saturation risks (CPU and disk IO).

2. Assuming we want to use N reindexing sessions, build the full list of indexes, with the table names they belong to,
   and "assign" each table to a particular reindexing session. See the query below that does it.

3. Using this "assignment", divide the whole list of indexes to N separate lists, so all the indexes for a particular
   table are present only in a single list – and now we can just run N sessions using these N lists.

For the step 2, here is a query that can help:

```sql
\set NUMBER_OF_SESSIONS 10

select
  format('%I.%I', n.nspname, c.relname) as table_fqn,
  format('%I.%I', n.nspname, i.relname) as index_fqn,
  mod(
    hashtext(format('%I.%I', n.nspname, c.relname)) & 2147483647,
    :NUMBER_OF_SESSIONS
  ) as session_id
from
  pg_index idx
  join pg_class c on idx.indrelid = c.oid
  join pg_class i on idx.indexrelid = i.oid
  join pg_namespace n on c.relnamespace = n.oid
where
  n.nspname not in ('pg_catalog', 'pg_toast', 'information_schema')
  -- and ... additional filters if needed
order by
  table_fqn, index_fqn;
```
