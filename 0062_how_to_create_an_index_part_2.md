Originally from: [tweet](https://twitter.com/samokhvalov/status/1729152164403249462), [LinkedIn post]().

---

# How to create an index, part 2

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

See also [Part 1](0061_how_to_create_an_index_part_1.md).

In part 1, we've covered the basics of how to build an index. Today we'll discuss parallelization and partitioning
aspects related to index creation. These two parts don't provide instructions on which index type to choose, when to use
partial indexes, indexes on expressions, or multi-column indexes – this will be covered in a separate howto.

## Long index creation and ways to speed it up

As already mentioned, too long index build time – say, hours – is not only inconvenient, it prevents `autovacuum` from
processing the table and also holds `xmin` horizon during the whole operation (which means that `autovacuum` cannot 
remove freshly-dead tuples in *all* tables in the database).

Thus, it is worth improving build time for individual indexes. General ideas:

1. Configuration tuning:

   - higher `maintenance_work_mem` as already discussed 
     > 🎯 **TODO:** show how with an experiment
   - checkpoint tuning: temporarily raised `max_wal_size` and `checkpoint_timeout` (doesn't require restart) reduces 
     checkpoint frequency, which may improve build time
     > 🎯 **TODO:**  an experiment to check it

2. Parallelization – use of multiple backends to speed up the whole operation.

3. Partitioning – splitting table to multiple physical tables reduces time needed to create an individual index.

## Parallel index build

The option `max_parallel_maintenance_workers` 
(PG11+; see [docs](https://postgresqlco.nf/doc/en/param/max_parallel_maintenance_workers/)) defines the maximum number
of parallel workers for` CREATE INDEX`. Currently, (as of PG16), it works only for building B-tree indexes.

The default `max_parallel_maintenance_workers` is `2`, and it can be raised, it doesn't require a restart; can be done
dynamically in session. The maximum depends on two settings:

- `max_parallel_workers`, default `8,` can be changed without a restart as well;
- `max_worker_processes`, default `8`, requires a restart to change.

Raising `max_parallel_maintenance_workers` can significantly decrease index build time, but this
should be done with proper analysis of CPU and disk IO utilization.

> 🎯 **TODO:**  experiment

## Indexes on partitioned tables

As was already discussed multiple times in other howtos, large tables (say, those exceeding 100 GiB; not a hard rule)
should be partitioned. Without it, if you have multi-terabyte tables, index creation will take very long time, during
which, `autovacuum` cannot process the table. This leads to higher levels of bloat.

It is possible to create indexes for individual partitions. However, it makes sense to consider using the unified
indexing approach for all partitions, and define an index on the partitioned table itself.

The `CONCURRENTLY` option cannot be used when creating an index on a partitioned table, it can only be used to index
individual partitions. However, this issue can be solved (PG11+):

1. Create indexes on all partitions separately, with `CONCURRENTLY`:

   ```
   create index concurrently i_p_123 on partition_123 ...;
   ```

2. Then create an index on the partitioned table (parent), without `CONCURRENTLY`, and also using the keyword `ONLY` – 
   it will be fast since this is not a large table, physically, but it will be marked `INVALID` until the next step is 
   fully executed:

   ```
   create index i_p_main on only partitioned_table ...;
   ```

3. Then, for every index on individual partitions, mark it as "attached" to the "main" index, using this slightly odd
   syntax (note that here we use index names, not table names):

   ```
   alter index i_p_main attach partition i_p_123;
   ```

Docs: 
[Partition Maintenance](https://postgresql.org/docs/current/ddl-partitioning.html#DDL-PARTITIONING-DECLARATIVE-MAINTENANCE).
