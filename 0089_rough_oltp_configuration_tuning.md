Originally from: [tweet](https://twitter.com/samokhvalov/status/1739559328099746245), [LinkedIn post]().

---

# Rough configuration tuning (80/20 rule; OLTP)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

The 80/20 rule (a.k.a. [Pareto principle](https://en.wikipedia.org/wiki/Pareto_principle)) is often enough to achieve a
good level of performance for OLTP workloads – it is recommended, in most cases, to start with this approach and focus
on query tuning. Especially if your system is rapidly changing – fighting for "the last 20%" just using configuration
tuning, when an overlooked schema-level optimization (e.g., a missing index) can "kill" the performance, making little
sense. However, those 20% make a lot of sense (e.g., budget-wise) to fight for, if workload and DB don't change too
fast, or if you have a lot (say, thousands) of Postgres nodes.

That's why simple empirical tuning services such as [PGTune](https://pgtune.leopard.in.ua) are often good enough. Here
we consider an example: a server of moderate size (64 vCPUs, 512 GiB RAM) serving moderate OLTP (web/mobile apps)
workloads.

The settings below should be considered as starting points and the values as only as rough guidelines – review for your
particular case, verify with experiments in non-production, and monitor all the changes closely.

Good resources:

- [PGTune](https://pgtune.leopard.in.ua)
- [postgresql.conf configurations](https://postgresqlco.nf)
- [postgresql_cluster's defaults](https://github.com/vitabaks/postgresql_cluster/blob/master/vars/main.yml)

1) `max_connections = 200`

    This should be based on the expected concurrent connections. Here we assume that we use connection pooling, reducing the
    need to keep a lot of idle connections. It is ok to set it to a higher value when running PG15+ (see: 
    [Improving Postgres Connection Scalability: Snapshots](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/improving-postgres-connection-scalability-snapshots/ba-p/1806462#conclusion-one-bottleneck-down-in-pg-14-others-in-sight)).

2) `shared_buffers = 128GB`

    Generally, setting this to about 25% of total memory is recommended. This is where Postgres caches table and index data.

    25% is considered as the most popular approach. Although it is sometimes criticized as not optimal, in most cases it
    is "good enough" and very safe.

3) `effective_cache_size = 384GB`

    This advises the planner on how much memory is available for caching data, including the OS cache.

4) `maintenance_work_mem = 2GB`

    Increases performance of maintenance operations such as `VACUUM`, `CREATE INDEX`, etc.

5) `checkpoint_completion_target = 0.9`

    Controls the aim for completing checkpoints spreading write activity, minimizing IO spikes.

6) `random_page_cost = 1.1`

    Fine-tune this to reflect the true cost of random IO. By default, it's 4, and `seq_page_cost` is 1 – this is ok for
    rotational disks. For SSDs, it makes sense to use equal or close values (Crunchy Data did 
    [some benchmark](https://twitter.com/brandur/status/1720477470116422028) work recently
    and found that 1.1 is slightly better than 1.0).

7) `effective_io_concurrency = 200`

    For SSDs, this can be set higher than for HDDs, reflecting the ability to handle more IO operations concurrently.

8) `work_mem = 100MB`

    Memory for sorts and joins per query. Set carefully, as high values may lead to out-of-memory issues if too many queries
    run concurrently.

9) `huge_pages = try`

    Using huge pages can improve performance by reducing page management overhead.

10) `max_wal_size = 10GB`

    This is a part of checkpoint tuning. 10GB is quite a large value; however, some may prefer using even larger, which
    presents a trade-off:

    - larger value help handle heavy writes better (lower IO stress), but
    - larger values also lead to longer recovery time in case of crashes.

    > 🎯 **TODO:** a separate howto on checkpoint tuning.

11) `max_worker_processes = 64`

    Maximum number of processes for the database cluster. Corresponds to the number of CPU cores.

12) `max_parallel_workers_per_gather = 4`

    The maximum number of workers that can be started by a single Gather or Gather Merge node.

13) `max_parallel_workers = 64`

    Total number of workers available for parallel operations.

14) `max_parallel_maintenance_workers = 4`

    Controls the number of workers for parallel maintenance tasks like index creation.

15) `jit = off`

    Turn it off for OLTP.

16) `Timeout settings`

    > 🎯 **TODO:** a separate howto

    ```
    statement_timeout = 30s
    idle_in_transaction_session_timeout = 30s
    ```

17) Autovacuum tuning

    > 🎯 **TODO:** a separate howto
    
    ```
    autovacuum_max_workers = 16
    autovacuum_vacuum_scale_factor = 0.01
    autovacuum_analyze_scale_factor = 0.01
    autovacuum_vacuum_insert_scale_factor = 0.02
    autovacuum_naptime = 1s
    # autovacuum_vacuum_cost_limit – increase if disks are powerful
    autovacuum_vacuum_cost_delay = 2
    ```

18) Observability, logging

    > 🎯 **TODO:** a separate howto
    
    ```
    logging_collector = on
    log_checkpoints = on
    log_min_duration_statement = 500ms # review
    log_statement = ddl
    log_autovacuum_min_duration = 0 # review
    log_temp_files = 0 # review
    log_lock_waits = on
    log_line_prefix = %m [%p, %x]: [%l-1] user=%u,db=%d,app=%a,client=%h
    log_recovery_conflict_waits = on 
    track_io_timing = on # review
    track_functions = all
    track_activity_query_size = 8192
    ```
