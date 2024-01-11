Originally from: [tweet](https://twitter.com/samokhvalov/status/1718135635448660118), [LinkedIn post]().

---

# How to speed up bulk load

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

If you need to load a lot of data, here are the tips that can help you do it faster.

## 1) COPY

Use `COPY` to load data, it's optimized for bulk load.

## 2) Less frequent checkpoints

Consider increasing `max_wal_size` and `checkpoint_timeout` temporarily.

Changing them does not require restart.

Increased values lead to increased recovery time in case of failure, but benefit is that checkpoints occur less often,
therefore:
1. less stress on disk,
2. less WAL data is written, thanks to decreased number of full page writes of the same pages (when load happens with
existing indexes).

## 3) Larger buffer pool

Increase `shared_buffers`, if you can.

## 4) No (or fewer) indexes

If load happens into a new table, create indexes after data load. When loading into an existing table,
[avoid over-indexing](0018_over_indexing.md).

Every additional index will significantly slow down the load.

## 5) No (or fewer) FKs and triggers

Similarly to indexes, foreign key constraints and triggers may significantly slow down data load – consider (re)creating
them after the bulk load.

Triggers can be disabled via `ALTER TABLE … DISABLE TRIGGERS ALL` – however, if triggers support some consistency
checks, you need to make sure that those checks are not violated (e.g., run additional checks after data load). FKs are
implemented via implicit triggers, and `ALTER TABLE … DISABLE TRIGGERS ALL` disables them too – loading data in this
state should be done with care.

## 6) Avoiding WAL writes

If this is a new table, consider completely avoiding WAL writes during the data load. Two options (both have limitations
and require understanding that data can be lost if a crash happens):

- Use unlogged table: `CREATE UNLOGGED TABLE …`. Unlogged tables are not archived, not replicated, they are not persistent (though, they survive normal restarts). However, converting an unlogged table to a normal one takes time (likely, a lot – worth testing), because he data needs to be written to WAL. More about unlogged tables in [this post](https://crunchydata.com/blog/postgresl-unlogged-tables); also, see [this StackOverflow discussion](https://dba.stackexchange.com/questions/195780/set-postgresql-table-to-logged-after-data-loading/195829#195829).

- Use `COPY` with `wal_level ='minimal'`. `COPY` has to be executed inside the transaction that created the table.
  In this case, due to `wal_level ='minimal'`, `COPY` writes won't be written to WAL
  (as of PG16, this is so only if table is unpartitioned).
  Additionally, consider using `COPY (FREEZE)` – this approach also provides a benefit: all tuples
  are frozen after the data load. Setting `wal_level='minimal'`, unfortunately, requires a restart, and additional
  changes (`archive_mode = 'off'`, `max_wal_senders = 0`). Of course, this method doesn't work well in most of the
  production cases, but can be good for single-server setups. Details for the `wal_level='minimal'` + `COPY (FREEZE)`
  recipe in [this post](https://cybertec-postgresql.com/en/loading-data-in-the-most-efficient-way/).

## 7) Parallelization

Consider parallelization. This may or may not speed up the process, depending on the bottlenecks of the single-threaded
process (e.g., if single-threaded load saturates disk IO, parallelization won't help). Two options:

- Partitioned tables and loading into multiple partitions using multiple workers
  ([Day 20: pg_restore tips](0020_how_to_use_pg_restore.md)).

- Unpartitioned table and loading in big chunks. Such chunks require preparation of them – it can be CSV split into
  pieces, or exported ranges of table data using multiple synchronized `REPEATABLE READ` transactions (working with the
  same snapshot via `SET TRANSACTION SNAPSHOT`; see [Day 8: How to speed up pg_dump](0008_how_to_speed_up_pg_dump.md).

If you use TimescaleDB, consider [timescaledb-parallel-copy](https://github.com/timescale/timescaledb-parallel-copy).

Last but not least: after a massive data load, don't forget to run `ANALYZE`.
