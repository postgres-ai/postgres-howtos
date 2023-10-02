Originally from: [tweet](https://twitter.com/samokhvalov/status/1708692950006612277), [LinkedIn post](...). 

---

# How to work with pg_stat_statements, part 2
Previous post: [0004_pg_stat_statements_part_1.md](./0004_pg_stat_statements_part_1.md).

Yesterday we discussed some basics of working with pgss, and the first set of derived metrics, `dM/dt` – time-based differentiation. Today we'll focus on the second set: `dM/dc`, where `c` is the number of calls (column `calls` in pgss). 

## Derivative 2. Calls-based differentiation
This set of metrics is not less important than time-based differentiation because it can provide you systematic view on characteristics of your workload and be a good tool for macro-optimization of query performance.

The metrics in this set help us understand the characteristics of a query, *on average*, for each query group.

Unfortunately, many monitoring systems disregard this kind of derived metrics. A good system has to present all or at least most of them, showing graphs how these values change over time (`dM/dc` time series).

Obtaining results for derived metrics of this kind is pretty straightforward:
- calculate difference of values `M` (the metric being studied) between two pgss snapshots: `M2 - M1`
- then, instead of using timestamps, get the difference of the "calls" values: `c2 - c1`
- then get `(M2 - M1) / (c2 - c1)`

Let's consider the meanings of various derived metrics obtained in such way:

1. `dM/dc`, where `M` `is calls` – a degenerate case, the value is always 1 (number of calls divided by the same number of calls).

2. `dM/dc`, where `M` is `total_plan_time + total_exec_time` – average query duration time in particular pgss group, a critically important metric for query performance observability. It can also be called "query latency". When applied to the aggregated value for all normalized queries in pgss, its meaning is "average query latency on the server" (with two important comments that pgss doesn't track failing queries and sometimes can have skewed data due to the `pg_stat_statements.max` limit). The main cumulative statistics system in Postgres doesn't provide this kind of information – `pg_stat_database` tracks some time metrics, `blk_read_time` and `blk_write_time` if `track_io_timing` is enabled, and, in PG14+, `active_time` – but it doesn't have information about the number of statements (!), only the number for transactions, `xact_commit` & `xact_rollback`, is present; in some cases, we can obtain this data from other sources – e.g., pgbench reports it if we use it for benchmarks, and pgBouncer reports stats for both transaction and query average latencies, but in general case, in observability tools, pgss can be considered as the most generic way get the query latency information. The importance of it is hard to overestimate – for example:
    - If we know that normally the avg query duration is <1 ms, then any spike to 10ms should be considered as a serious incident (if it happened after a deployment, this deployment should be reconsidered/reverted). For troubleshooting, it also helps  to apply segmentation and determine which particular query groups contributed to this latency spike – was it all of them or just particular ones?
    - In many cases, this can be taken as the most important metric for large load testing, benchmarks (for example: comparing average query duration for PG 15 vs. PG 16 when preparing for a major upgrade to PG 16).

3. `dM/dc`, where `M `is `rows` – average number of rows returned by a query in a given query group. For OLTP cases, the groups having large values (starting at a few hundreds or more, depending on the case) should be reviewed:
    - if it's intentional (say, data dumps), no action needed,
    - if it's a user-facing query and it's not related to data exports, then probably there is a mistake such as lack of `LIMIT` and proper pagination applied, then such queries should be fixed.

4. `dM/dc`, where `M` is `shared_blks_hit + shared_blks_read` – average number of  "hits + reads" from the buffer pool. It is worth translating this to bytes: for example, `500,000` buffer hits&reads translates to `500000 GiB * 8 / 1024 / 1024 =  ~ 3.8 GiB`, this is a significant number for a single query, especially if its goal is to return just a row or a few. Large numbers here should be considered as a strong call for query optimization. Additional notes:
    - in many cases, it makes sense to have hits and reads can be also considered separately – there may be the cases when, for example, queries in some pgss group do not lead to high disk IO and reading from the page cache, but they have so many hits in the buffer pool, so their performance is suboptimal, even with all the data being cached in the buffer pool
    - to have real disk IO numbers, it is worth using https://github.com/powa-team/pg_stat_kcache
    - a sudden change in the values of this metric for a particular group that persists over time, can be a sign of plan flip and needs to be studied
    - high-level aggregated values are also interesting to observe, answering questions like "how many MiB do all queries, on average, read on this server?"

5. `dM/dc`, where `M` is `wal_bytes` (PG13+) – average amount of WAL generated by a query in the studied pgss group measured in bytes. It is helpful for identification of query groups that contribute most to WAL generation. A "global" aggregated value for all pgss records represents the average number of bytes for all statements on the server. Having graphs for this and for "`dM/dc`, where `M` is `wal_fpi`" can be very helpful in certain situations such as checkpoint tuning: with `full_page_writes = on`, increasing the distance between checkpoints, we should observe reduction of values in this area, and it may be interesting to study different particular groups in pgss separately.

===

Tomorrow, we'll finish with step 3 of this pgss-related how-to.