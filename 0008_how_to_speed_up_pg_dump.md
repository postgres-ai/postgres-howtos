Originally from: [tweet](https://twitter.com/samokhvalov/status/1709490130749378796), [LinkedIn post](https://www.linkedin.com/pulse/how-speed-up-pgdump-when-dumping-large-postgres-nikolay-samokhvalov/).

---

# How to speed up pg_dump when dumping large databases

<img src="files/0008_cover.png" width="600" />

After a series of rather hefty posts, let's take a breather with some lighter material.

Today we'll discuss how to speed up `pg_dump` when dealing with large data volumes.

Speeding up options discussed here:
1. Compression
2. Dump/restore without saving dumps on disk
3. Parallelized `pg_dump`
4. Advanced custom parallelization

## Option 1: Consider applying compression
In the cases of weak disks or network, it makes sense to apply compression. If the `plain` (default) format is used, then you can just use `pg_dump ... | gzip`. Note that this is not going to help to speed the process up if disk IO and network (if it's used) are not saturated (inspect resource utilization with standard tools like `iostat`, `top`, and so on).

## Option 2: Avoid dumping to disk, restore on the fly
When the `directory` format is used (option `-Fd`; this format is the most flexible and I usually use it unless I have a specific situation), then compression is applied by default (`gzip` by default, also available `lz4` and `zstd`).

It also makes sense to use `pg_dump -h ... | pg_restore` and avoid writing to disk and restoring the dump "on the fly". Unfortunately, this can be done only when pg_dump is creating a `plain` dump – with `directory` format, it's not working. To solve this problem, there is a 3rd-party tool called [pgcopydb](https://github.com/dimitri/pgcopydb).

## Option 3: pg_dump -j$N
For servers with a high number of CPUs, when you deal with multiple tables and create dumps in the `directory` format, parallelization (option `-j$N`) can be very helpful. A single, but partitioned, table is going to behave similarly to multiple tables – because physically, the dumping will be applied to multiple tables (partitions).

Consider a standard table `pgbench_accounts`, created by [pgbench](https://postgresql.org/docs/current/pgbench.html), partitioned into 30 partitions and relatively large, 100M rows – the size of this data (without indexes) is `~12 GiB`:
```
❯ pgbench -i -s1000 --partitions=30 test
dropping old tables...
creating tables...
creating 30 partitions...
generating data (client-side)...
100000000 of 100000000 tuples (100%) done (elapsed 53.64 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 146.30 s (drop tables 0.15 s, create tables 0.22 s, client-side generate 54.02 s, vacuum 25.19 s, primary keys 66.71 s).
```

The size of data in the database `test` is now `~12 GiB` and the majority of it is in the shared table `pgbench_accounts`.

The most flexible format of dumps is `directory` – option `-Fd`. Measuring time a few times to make sure caches are warmed up (macbook m1, 16 GiB RAM, PG15, `shared_buffers='4GB'`):
```
❯ time pg_dump -Fd -f ./test_dump test
pg_dump -Fd -f ./test_dump test  45.94s user 2.65s system 77% cpu 1:02.50 total
❯ rm -rf ./test_dump
❯ time pg_dump -Fd -f ./test_dump test
pg_dump -Fd -f ./test_dump test  45.83s user 2.69s system 79% cpu 1:01.06 total
```

– `~61 sec`.

Speeding up with `-j8` to have 8 parallel `pg_dump` workers:
```
❯ time pg_dump -Fd -j8 -f ./test_dump test
pg_dump -Fd -j8 -f ./test_dump test  57.29s user 6.02s system 259% cpu 24.363 total
❯ rm -rf ./test_dump
❯ time pg_dump -Fd -j8 -f ./test_dump test
pg_dump -Fd -j8 -f ./test_dump test  57.59s user 6.06s system 261% cpu 24.327 total
```

– `~24 sec` (vs. `~61 sec` above).

## Option 4: Advanced parallelization for large unpartitioned tables
When dumping a database where there is a single unpartitioned table that is much larger than other tables (e.g., a huge unpartitioned "log" table containing some historical data), standard pg_dump parallelization, unfortunately, won't help. Using the same example as above, but without partitioning:
```
❯ pgbench -i -s1000 test
dropping old tables...
creating tables...
generating data (client-side)...
100000000 of 100000000 tuples (100%) done (elapsed 51.71 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 116.23 s (drop tables 0.73 s, create tables 0.03 s, client-side generate 51.93 s, vacuum 4.41 s, primary keys 59.14 s).
❯
❯ rm -rf ./test_dump
❯ time pg_dump -Fd -j8 -f ./test_dump test
pg_dump -Fd -j8 -f ./test_dump test  48.24s user 3.25s system 83% cpu 1:01.83 total
```

– multiple workers are not really helpful here, because most of the time, a single worker is going to work (dumping the largest unpartitioned table).

This is because `pg_dump` parallelization is working at table level and cannot parallelize dumping a single table.

To parallelize dumping a single large table, a custom solution is needed. To do that, we need to use multiple SQL clients such as psql, each one working with transaction at `REPEATABLE READ` isolation level (`pg_dump` is also using this level when working; see [the docs](https://postgresql.org/docs/current/transaction-iso.html)), and (important!) all of the dumping transactions need to use the same snapshot.

The process can be as follows:
1. In one connection (e.g., in one `psql` session), start a transaction at the `REPEATABLE READ` level:
    ```
    test=# start transaction isolation level repeatable read;
    START TRANSACTION
    ```
2. This transaction must be open till the very end of the process – we need to make sure it is so.
3. In the same session, use the function `pg_export_snapshot()` to obtain the snapshot ID:
    ```
    test=*# select pg_export_snapshot();
     pg_export_snapshot
    ---------------------
     00000004-000BF714-1
    (1 row)
    ```
4. In other sessions, we also open `REPEATABLE READ` transactions and set them to use the very same snapshot (of course, we run all of them in parallel, this is the whole point of speeding up):
    ```
    test=# start transaction isolation level repeatable read;
    START TRANSACTION
    test=*# set transaction snapshot '00000004-000BF714-1';
    SET
    ```
5. Then in each session, we dump a chunk of the large table, making sure the access method is fast (`Index Scan`; alternatively, for example when no proper indexes are present, you can use the ranges of hidden column `ctid` and benefit from using [TID Scan](https://www.pgmustard.com/docs/explain/tid-scan), avoiding `Seq Scan`). For example, dumping the chunk 1 of 8 for `pgbench_accounts`:
    ```
    test=# start transaction isolation level repeatable read;
    START TRANSACTION
    test=*# set transaction snapshot '00000004-000BF714-1';
    SET
    test=*# copy (select * from pgbench_accounts where aid <= 12500000) to stdout;
    ```
6. To dump other, smaller tables, we can involve `pg_dump` - it also supports working with particular snapshots, via option `--snapshot=...`. In this case, we need to exclude the data for our large table using `--exclude-table-data=...` because we take care of it separately. In this case, we can also involve parallelization. For example:
    ```
    ❯ pg_dump \
      -Fd \
      -j2 \
      -f ./everything_but_pgba_data.dump \
      --snapshot="00000004-000BF714-1" \
      --exclude-table-data="pgbench_accounts" \
      test
    ```
7. Do not forget to close the first transaction when everything is done – long-running transaction are harmful for OLTP workloads.
8. To restore, we need to follow the usual pg_dump order: DDL defining objects except indexes; then data load; and finally, constraint validation and index creation. For this, we can benefit from having dump in the `directory` format and use `pg_restore`'s options `-l` and `-L` to list the objects in the dump and filter them to restore, respectively.

A good post about dealing with snapshots when making database dumps: ["Postgres 9.5 feature highlight - pg_dump and external snapshots"](https://paquier.xyz/postgresql-2/postgres-9-5-feature-highlight-pg-dump-snapshots/). A very interesting additional consideration in that post is related to a special case of dumping: initialization of logical replicas. It is possible to use custom dumping methods synchronized with the position of logical slot, but creation of such slot has to be done via replication protocol (`CREATE_REPLICATION_SLOT foo3 LOGICAL test_decoding;`), not using SQL (`select * from  pg_create_logical_replication_slot(...);`).

---

That's it for today. Good dumping speeds in production (2+ TiB/h) to everyone!
