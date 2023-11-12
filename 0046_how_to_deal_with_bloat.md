Originally from: [tweet](https://twitter.com/samokhvalov/status/1723333152847077428), [LinkedIn post]().

---

# How to deal with bloat

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## What is bloat?

Bloat is the free space inside pages, created when `autovacuum` deletes a large number of tuples.

When a row in a table is updated, Postgres doesn't overwrite the old data. Instead, it marks the old row version (tuple)
as "dead" and creates a new row version. Over time, as more rows are updated or deleted, the space taken up by these
dead tuples can accumulate. At some point, `autovacuum` (or manual `VACUUM`) deletes dead tuples, leaving free space
inside pages, available for reuse. But if large numbers of dead tuples accumulate, large volumes of free space can be
left behind – in the worst cases, it can occupy 99% of all table or index space, or even more.

Low values of bloat (say, below 40%) should not be considered a problem, while high values definitely should be, as they
lead to bad consequences:

1. Higher disk usage
2. More IO needed for read and write queries
3. Lower cache efficiency (both buffer pool and OS file cache)
4. As a result, worse query performance

## How to check bloat

Index and table bloat should be regularly checked. Note that most queries that are commonly used are estimation-based
and are prone to false positives -- depending on table structure, it can show some non-existent bloat (I saw cases with
up to 40% of phantom bloat in freshly created tables). But such queries are fast and don't require additional extensions
installed. Examples:

- [Estimated table bloat](https://github.com/NikolayS/postgres_dba/blob/master/sql/b1_table_estimation.sql)
- [Estimated btree index bloat](https://github.com/NikolayS/postgres_dba/blob/master/sql/b2_btree_estimation.sql)

Recommendations for use in monitoring systems:

- There is little sense in running them often (e.g., every minute), as bloat levels don't change rapidly.
- There should be a warning provided to users that the results are estimates.
- For large databases, query execution may take a long time, up to many seconds, so the frequency of checks and
  `statement_timeout` might need to be adjusted.

Approaches to determine the bloat levels more precisely:

- queries based on `pgstattuple` (the extension has to be installed)
- checking DB object sizes on a clone, running `VACUUM FULL` (heavy and blocks queries, thus not for production), and
  then checking sizes again and comparing before/after

Periodical checks are definitely recommended to control bloat levels and react, when needed.

## Index bloat mitigation (reactive)

Unfortunately, in databases that experience many `UPDATE`s and `DELETE`s, index health inevitably degrades over time.
This means, that indexes need to be rebuilt regularly.

Recommendations:

* Use `REINDEX CONCURRENTLY` to rebuild bloated index in a non-blocking fashion.
* Remember that `REINDEX CONCURRENTLY` holds the `xmin` horizon when running. This affects `autovacuum`'s ability to
  clean up freshly-dead tuples in all tables and indexes. This is another reason to use partitioning; do not allow
  tables to exceed certain threshold (say, more than 100 GiB).
* You can monitor the progress of reindexing using the approach
  from [Day 15: How to monitor CREATE INDEX / REINDEX](0015_how_to_monitor_index_operations.md).
* Prefer using Postgres versions 14+, since in PG14, btree indexes were significantly optimized to degrade much slower
  under written workloads.

## Table bloat mitigation (reactive)

Some levels of table bloat may be considered as not a bad thing because they increase chances of having optimized
`UPDATE`s -- HOT (Heap-Only Tuples) `UPDATE`s.

However, if the level is concerning, consider using [pg_repack](https://github.com/reorg/pg_repack) to rebuild the table
without long-lasting exclusive locks. Alternative to `pg_repack`:
[pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze).

Normally, this process doesn't need to be scheduled and fully automated; usually, it is enough to apply it under control
only when high table bloat is detected.

## Proactive bloat mitigation

* Tune `autovacuum`.
* Monitor the `xmin` horizon and don't allow it to be too far in the past --
  [Day 45: How to monitor xmin horizon to prevent XID/MultiXID wraparound and high bloat](0045_how_to_monitor_xmin_horizon.md).
* Do not allow unnecessary long-running transactions (e.g., > 1h), neither on the primary, nor on standbys with
  `hot_standby_feedback` turned on.
* If on Postgres 13 or older, consider upgrading to 14+ to benefit from btree index optimizations.
* Partition large (100+ GiB) tables.
* Use partitioning for tables with queue-like workloads, even if they are small, and use `TRUNCATE` or drop partitions
  with old data instead of using `DELETE`s; in this case, vacuuming is not needed, bloat is not an issue.
* Do not use massive `UPDATE`s and `DELETE`s, always work in batches (lasting not more than 1-2s);
  ensure that `autovacuum` cleans up dead tuples promptly or `VACUUM` manually when massive data changes need to happen.

## Materials worth reading

* Postgres docs: [Routine Vacuuming](https://postgresql.org/docs/current/routine-vacuuming.html)
* [When `autovacuum` does not vacuum](https://2ndquadrant.com/en/blog/when-`autovacuum`-does-not-vacuum/)
* [How to Reduce Bloat in Large PostgreSQL Tables](https://timescale.com/learn/how-to-reduce-bloat-in-large-postgresql-tables/)
