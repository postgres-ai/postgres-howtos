Originally from: [tweet](https://twitter.com/samokhvalov/status/1710176204953919574), [LinkedIn post](...). 

---

# How to find query examples for problematic pg_stat_statements records

// I post a new PostgreSQL "howto" article every day. Join me in this journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

In a few hours, I'm presenting my "Seamless Postgres query optimization" tutorial at DjangoCon US, so it's a good time to talk about transitioning from `pg_stat_statements` (`pgss`) to `EXPLAIN`.

## The problem of jumping from pgss to EXPLAIN

Once a problematic `pgss` record is identified (which is the subject of query macro-optimization that we've discussed on [days 5-7](https://twitter.com/samokhvalov/status/1709069225762095258), the first thing to do is to understand the direction of optimization.

Two most common basic situations of `pgss` records requiring optimization:
1. If `calls` is very high (a lot of QPS, queries per second), the main method to optimize is reduction of this number – this is to be done on client (app) side.
2. If `mean_exec_time + mean_plan_time` is high (for the OLTP context – web and mobile apps – `100ms` should be [considered slow](https://postgres.ai/blog/20210909-what-is-a-slow-sql-query)), this is when we need to apply query micro-optimization – use `EXPLAIN`.

Of course, it's also not uncommon to have a combination of these two basic cases – quite frequent query with poor latency.

In this post, we won't discuss how to use `EXPLAIN` and `EXPLAIN (ANALYZE, BUFFERS)`. Instead, we'll focus on finding proper materials for `EXPLAIN` – particular query examples that need to be studied and improved.

It is worth remembering that a single `pgss` records can be associated with individual queries that are executed differently – using different plans. A basic example illustrating it:
```
nik=# create table t1 as select 1::int8 as c1;
SELECT 1
nik=# insert into t1 select 2
from generate_series(1, 1000000);
INSERT 0 1000000
nik=# create index on t1(c1);
CREATE INDEX
nik=# vacuum analyze t1;
VACUUM
nik=# explain select from t1 where c1 = 1;
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Only Scan using t1_c1_idx on t1  (cost=0.42..4.44 rows=1 width=0)
   Index Cond: (c1 = 1)
(2 rows)

nik=# explain select from t1 where c1 = 2;
                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..16981.01 rows=1000001 width=0)
   Filter: (c1 = 2)
(2 rows)
```

– both queries here will be registered as `select * from t1 where c1 = $1` in `pgss`. But plans are different, because for `c1 = 1`, we have high selectivity, while for `c1 = 2` it is really bad (targeting all but 1 rows in the table).

This means that looking at just `pgss` record demonstrating poor query latency, we cannot quickly jump to using `EXPLAIN` – we need to find particular query samples to work with. 

Below, we discuss options to solve this problem.

## Option 1: guessing
In some cases, it may be fine to guess. But, I had really bad cases when I lost a lot of time making a mistake with guessing. For example, in one case, dealing with a boolean column, I decided to use the value that had a very bad selectivity, and spent a lot of time optimizing this situation, before realizing that application code never ever is going to use it.

It might be tempting to use `pg_statistic` to improve the guesswork. But unfortunately, in general case, this doesn't work really well, because of lack of multi-column statistic (except when it's created explicitly) – without it, we're going to have unrealistic parameter variants in lots of cases.

So this method is limited and can be used only for simple cases.

## Option 2: get examples from Postgres log
It is possible to find examples in the Postgres log – of course, if they are logged (usually via the `log_min_duration_statement` parameter or the `auto_explain` extension). To find examples for a given `pgss` record, we need to be able to find association of logged queries and `pgss` records. Two options:

1. For PG14+, option [compute_query_id](https://postgresqlco.nf/doc/en/param/compute_query_id/) can provide the same queryid value that is used in pg_stat_statements, to the log entry. 
2. Alternatively, we can use an excellent library [libpg_query](https://github.com/pganalyze/libpg_query; Ruby, Go, Python and other options are also available). It can be applied both to normalized (`pgss` records) and individual queries, producing so-called fingerprint, that can be then used to find the relationships we need.

In general, using Postgres logs to find query examples is a good method, but for heavily-loaded systems, where it is impossible to log all queries, it is going to supply us with very slow examples only – those that exceed [log_min_duration_statatement](https://postgresqlco.nf/doc/en/param/log_min_duration_statement/) (usually set to some quite high value, e.g. `500ms`).

This situation can be improved with sampling and lowering the threshold for slow queries or even getting rid of it completely. Parameters for it:
- [log_min_duration_sample](https://postgresqlco.nf/doc/en/param/log_min_duration_sample/) (PG13+) 
- [log_statement_sample_rate](https://postgresqlco.nf/doc/en/param/log_statement_sample_rate/) (PG13+)
- [log_transaction_sample_rate](https://postgresqlco.nf/doc/en/param/log_transaction_sample_rate/) (PG12+)

Alternatively, `auto_explain` can be used, it also supports sampling (`auto_explain.sample_rate`, all currently supported versions), and it can look really attractive since it brings plans as they were right on production. Installation of `auto_explain` should be done with careful testing and analysis of overhead.

## Option 3: sample queries from pg_stat_activity
This method can be attractive since it doesn't require us to turn on the expensive logging of too many queries. And even low-latency queries, if they are frequent enough, are going to be eventually captured if we observe `pg_stat_activity.query` long enough.

However, there are two important limitations here.

First, the column pg_stat_statements.query_id, useful to connect samples from `pg_stat_activity` (`pgsa`) with `pgss` records, was added relatively recently, in PG14. For older versions, we would end up using some regular expressions (implementation can be cumbersome/fragile) of libpg_query's fingerprints (meaning that we need to sample all `pgsa` records and then post-process them). So this method is better to use in PG14+.

Second, by default, `pg_stat_activity.query` is truncated to 1024 characters – this is defined by [track_activity_query_size](https://postgresqlco.nf/doc/en/param/track_activity_query_size/), which is 1024 by default. It is recommended to increase it significantly – e.g., to 10k, to allow larger queries to be sampled and analyzed. Unfortunately, changing this setting requires a restart.

## Option 4: eBPF
This option is not yet fully developed, but there is an opinion that it can be a very good alternative in the future: use eBPF for either sampling queries (alternative to `pgsa`), or even for sampling queries + normalizing them (being alternative to both `pgsa` and `pgss`).

So – TBD. Meanwhile, check out these interesting resources:
- pgtracer https://github.com/Aiven-Open/pgtracer (presentation on PostgresTV: https://youtube.com/watch?v=tvJgMV-8nfU)
- Analyzing Postgres performance problems using perf and eBPF – a talk video where Andres Freund describes how to combine pg_stat_statements + BPF  https://youtu.be/HghP4D72Noc?si=tFuQuDWKrScJ8w2i&t=1389 (code: https://github.com/anarazel/pg-bpftrace)

## Option 5: generic plans

TBD:
- new feature in pg16's EXPLAIN
- tricks for versions <16

## Summary
- In PG14+, use `compute_query_id` to have quer`y_id values both in Postgres logs and `pg_stat_activity`
- Increase `track_activity_query_size` (requires restart) to be able to track larger queries in `pg_stat_activity`
- Organize workflow to combine records from `pg_stat_statements` and query examples from logs and `pg_stat_activity`, so when it comes to query optimization, you have good examples ready to be used with `EXPLAIN (ANALYZE, BUFFERS)`.

As a reminder, once you have good examples, the best (fastest&cheapest) way to verify optimization ideas is to use properly tuned thin clones – check out [DBLab](https://twitter.com/Database_Lab).

---