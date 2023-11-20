Originally from: [tweet](https://twitter.com/samokhvalov/status/1725520009601142821), [LinkedIn post]().

---

# How to reduce WAL generation rates

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

In a fast-growing project, one of the important optimization vectors is reduction of the amount of WAL (Write Ahead Log)
generated.

## Why it is important

WAL is a core mechanism that serves as foundation for post-failure recovery, backups, and replication.

The more WAL is generated per second, the more data needs to be replicated and backed up per second, and, as a
consequence, various risks may grow as well: replication lags, WAL archiving lags, and post-failure recovery duration.

## How to measure WAL generation rates

When a new transaction creates a new WAL record, it receives LSN (Log Sequence Number). Monitoring of the current LSN
position is straightforward:

```sql
select pg_current_wal_lsn();
```

This metric should be present in any Postgres monitoring.

The difference between two LSNs is the number of bytes between two positions, and the math can be done in Postgres if
needed – using the `pg_lsn` data type:

```sql
nik=# select pg_size_pretty('3/ED5F1E0'::pg_lsn - '0/110A1E0');
 pg_size_pretty
----------------
 12 GB
(1 row)
```

If your monitoring doesn't have it, you can understand how much of WAL data was generated per hour or day by looking at:

- `pg_wal` directory to see the WAL file names
- inspecting the backups (for example, checking the names of two full backups created by
  WAL-G: `wal-g backup-list --detail`)

Both methods should help you to get two LSN values corresponding to two distant points of time.

For further details,
see [Day 9: How to understand the LSN values and WAL file name](0009_lsn_values_and_wal_filenames.md).

## WAL metrics in query analysis

Since Postgres 13, both `pg_stat_statements` and `EXPLAIN` can provide WAL-related metrics:

1) `pg_stat_statements`: metrics `wal_records`, `wal_fpi`, `wal_bytes`
   ([docs](https://postgresql.org/docs/current/pgstatstatements.html)). A basic analysis example:

   ```sql
   with time_period(delta_sec) as (
     select extract(epoch from now() - stats_reset)
     from pg_stat_statements_info
   )
   select
     now(),
     delta_sec,
     round(wal_bytes / delta_sec) as wal_bytes_per_sec,
     round(wal_bytes / calls) as wal_bytes_per_call,
     round(total_exec_time::numeric / delta_sec, 2) as exec_ms_per_sec,
     round(mean_exec_time::numeric, 2) as exec_ms_per_call,
     queryid
   from
     pg_stat_statements,
     time_period
   order by wal_bytes desc 
   limit 25;
   ```

2) `EXPLAIN`: use `explain (analyze, buffers, wal)` to see the WAL metrics in the execution plan

   ```sql
   nik=# explain (analyze, buffers, wal) insert into t select i from generate_series(1, 100000) as i;
                                                                QUERY PLAN
   -------------------------------------------------------------------------------------------------------------------------------------
    Insert on t  (cost=0.00..1000.00 rows=0 width=0) (actual time=159.378..159.378 rows=0 loops=1)
      Buffers: shared hit=100895 dirtied=442 written=442
      WAL: records=100000 fpi=1 bytes=5900343
      ->  Function Scan on generate_series i  (cost=0.00..1000.00 rows=100000 width=4) (actual time=26.179..30.696 rows=100000 loops=1)
    Planning Time: 1.945 ms
    Execution Time: 160.483 ms
   (6 rows)
   ```

## Full-page writes

The "fpi" metrics in both `pg_stat_statements` and `EXPLAIN` results show how many full-page inserts (full-page writes)
happened.

If configuration parameter `full_page_write` is `on` (it is so by default; check it: `show full_page_writes;`), then
after each checkpoint the very first change in a page leads to the whole page being written in WAL. By default, the page
size is 8 KiB and it is so in most Postgres installations (check it: `show block_size;`). This means that if only a very
small part of the page is changed, still the whole page needs to be written after checkpoint. Subsequent writes in the
same page are going to be normal (only changes are recorded to WAL), but once a new checkpoint happens, then again, a
new full-page write is needed first. More about it:

- Hironobu Suzuki's "The Internals of PostgreSQL". Chapter 9 "Write Ahead Logging – WAL", 
  [9.1.3. Full-Page Writes](https://interdb.jp/pg/pgsql09.html#_9.1.3).
- Egor Rogov's "PostgreSQL 14 Internals", "10.4 Recovery"
- Postgres wiki: [Full page writes](https://wiki.postgresql.org/wiki/Full_page_writes)
- Tomas Vondra,
  [On the impact of full-page writes (2016)](https://2ndquadrant.com/en/blog/on-the-impact-of-full-page-writes/)

## Optimization ideas

Below we discuss various ideas that can help you reduce the amount of WAL generated.

1) **Checkpoint tuning: increase distance between checkpoints**

   Increasing intervals between checkpoints gives two benefits, especially if workload contains many random (not
   sequential like during massive data load with `COPY`) writes:

    - fewer full page writes,
    - fewer repetitive flushes of the same buffer (that once flushed, might become dirty again too quickly due to new
      writes).

   To increase distance, we just need to increase `max_wal_size` (default `1GB`) and checkpoint_timeout
   (default `5min`). But this needs to be done with understanding of the trade-off: the bigger distance between
   checkpoints means more WALs will need to be replayed to achieve consistency point in various situations:

    - longer recover time after crashes,
    - longer time to provision new nodes from backups.

   Still, this method is a must-have for larger setups, since it gives substantial improvement.

   Changing `max_wal_size` and `checkpoint_timeout` doesn't require restart.

2) **Checkpoint tuning: enable compression of full-page writes**

   Consider `wal_compression` – compression of full-page writes. In most cases, it is worth doing it 
   (although, there are some reports that it led to higher CPU usage and decision to revert the change).

   Changing it doesn't require restart.

3) **Optimize queries**

   Using `pg_stat_statements` and `EXPLAIN`, as discussed above, find and optimize the most WAL-write-heavy queries.

   One of the approaches to optimize writes is to encourage Postgres to use more `HOT UPDATE`s (for that, we need to
   ensure that pages have free space – surprisingly, some bloat is helpful here – and we don't over-index the table so
   the columns we're changing do not participate in index definitions).

4) **Drop unused and redundant indexes**

   During query optimization, keep in mind that for non-`HOT UPDATE`s and `INSERT`s, amount of WAL produced depends on
   the number of indexes the table has. Index cleanup is a very helpful approach to reduce WAL volumes from such writes.

5) **Partitioning**

    Partitioning of large (100+ GiB) tables improves data locality for writes – for example, `UPDATE`s of a bunch of rows
    could be scattered among many pages if table is not partitioned, and with partitioning schema that defines old
    partitions (that receive almost no writes) and partitions with fresh data, most writes are going to be localized in the
    fresh partitions, which can help reduce WAL generation rates.
