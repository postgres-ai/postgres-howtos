Originally from: [tweet](https://twitter.com/samokhvalov/status/1708244676313317635), [LinkedIn post](...). 

---

# How to work with pg_stat_statments, part 1

There are two big areas of query optimization:
1. "Micro" optimization: analysis and improvement of particular queries. Main tool: `EXPLAIN`.
2. "Macro" optimization: analysis of whole or large parts of workload, segmentation of it, studying characteristics, going from top to down, to identify and improve the parts that behave the worst. Main tools: `pg_stat_statements` (and additions or alternatives), wait event analysis, and Postgres logs.

Today we focus on how to read and use `pg_stat_statements`, starting from basics and proceeding to using the data from it for macro optimization.

Docs: https://postgresql.org/docs/current/pgstatstatements.html

Extension `pg_stat_statements` (for short, "pgss") became standard de-facto for macro-analysis.

It tracks all queries, aggregating them to query groups – called "normalized queries" – where parameters are 
removed.

There are certain limitations, some of which are worth remembering:
- it doesn't show anything about ongoing queries (can be found in `pg_stat_activity`)
- a big issue: it doesn't track failing queries, which can sometimes lead to wrong conclusions (example: CPU and disk IO load are high, but 99% of our queries fail on `statement_timeout`, loading our system but not producing any useful results – in this case, pgss is blind)
- if there are SQL comments, they are not removed, but only the first comment value is going to be present in the `query` column for each normalized query

The view pg_stat_statements has 3 kinds of columns:
1. `queryid` – an identifier of normalized query. In the latest PG version it can also be used to connect (`JOIN`) data from pgss to pgsa (`pg_stat_statements`) and Postgres logs. Surprise: `queryid` value can be negative.
2. Descriptive columns: ID of database (`dbid`), user (`userid`), and the query itself (`query`).
3. Metrics. Almost all of them are cumulative: `calls`, `total_time`, `rows`, etc. Non-cumulative: `stddev_plan_time`, `stddev_exec_time`, `min_exec_time`, etc. In this post, we'll focus only on cumulative ones.

To read and interpret data from pgss, you need three steps:
1. Take two snapshots corresponding to two points of time.
2. Calculate the diff for each cumulative metric and for time difference for the two points in time
    - a special case is when the first point in time is the beginning of stats collection – in PG14+, there is a separate view, `pg_stat_statements_info`, that has information about when the pgss stats reset happened; in PG13 and older this info is not stored, unfortunately.
3. (the most interesting part!) Calculate three types of derived metrics for each cumulative metric diff – assuming that M is our metric and remembering some basics of calculus from high school:
    a. `dM/dt` – time-based differentiation of the metric `M`;
    b. `dM/dc` – calls-based differentiation (I'll explain it in detail in the next post);
    c. `%M` – percentage that this normalized query takes in the whole workload considering metric `M`.

Step 3 can be also applied not to particular normalized queries on a single host but bigger groups – for example:
- aggregated workload for all standby nodes
- whole workload on a node (e.g., the primary)
- bigger segments such as all queries from specific user or to specific database
- all queries of specific type – e.g., all `UPDATE` queries

If your monitoring system supports pgss, you don't need to deal with working with snapshots manually – although, keep in mind that I personally don't know any monitoring that works with pgss perfectly, preserving all kinds of information discussed in this post (and I studied quite a few of Postgres monitoring tools).

// Below I sometimes call normalized query "query group" or simply "group".

Let's mention some metrics that are usually most frequently used in macro optimization (full list: https://postgresql.org/docs/current/pgstatstatements.html#PGSTATSTATEMENTS-PG-STAT-STATEMENTS):
1. `calls` – how many query calls happened for this query group (normalized query)
2. `total_plan_time` and `total_exec_time` – aggregated duration for planning and execution for this group (again, remember: failed queries are not tracked, including those that failed on `statement_timeout`)
3. `rows` – how many rows returned by queries in this group
4. `shared_blks_hit` and `shared_blks_read` – number if hit and read operations from the buffer pool. Two important notes here:
    - "read" here means a read from the buffer pool – it is not necessarily a physical read from disk, since data can be cached in the OS page cache. So we cannot say these reads are reads from disk. Some monitoring systems make this mistake, but there are cases that this nuance is essential for our analysis to produce correct results and conclusions.
    - the names "blocks hit" and "blocks read" might be a little bit misleading, suggesting that here we talk about data volumes – number of blocks (buffers). While aggregation here definitely make sense, we must keep in mind that the same buffers may be read or hit multiple times. So instead of "blocks have been hit" it is better to say "block hits".
5. `wal_bytes` – how many bytes are written to WAL by queries in this group

There are many more other interesting metrics, it is worth exploring them all.

Once you obtained 2 snapshots of pgss (remembering timestamp when they were collected), let's consider practical meaning of the three derivatives we discussed:

Derivative 1. Time-based differentiation 

* `dM/dt`, where `M` is `calls` – the meaning is simple. It's QPS (queries per second). If we talk about particular group (normalized query), it's that all queries in this group have. `10,000` is pretty large so, probably, you need to improve the client (app) behavior to reduce it, `10` is pretty small (of course, depending on situation). If we consider this derivative for whole node, it's our "global QPS".

* `dM/dt`, where `M` is `total_plan_time + total_exec_time` – this is the most interesting and key metric in query macro analysis targeted at resource consumption optimization (goal: reduce time spent by server to process queries). Interesting fact: it is measured in "seconds per second", meaning: how many seconds our server spends to process queries in this query group. *Very* rough (but illustrative) meaning: if we have `2 sec/sec` here, it means that we spend 2 seconds each second to process such queries – we definitely would like to have more than 2 vCPUs to do that. Although, this is a very rough meaning because pgss doesn't distinguish situations when query is waiting for some lock acquisition vs. performing some actual work in CPU (for that, we need to involve wait event analysis) – so there may be cases when the value here is high not having a significant effect on the CPU load.

* `dM/dt`, where `M` is `rows` – this is the "stream" of rows returned by queries in the group, per second. For example, `1000 rows/sec` means a noticeable "stream" from Postgres server to client. Interesting fact here is that sometimes, we might need to think how much load the results produced by our Postgres server put on the application nodes – returning too many rows may require significant resources on the client side.

* `dM/dt`, where `M` is `shared_blks_hit + shared_blks_read` - buffer operations per second (only to read data, not to write it). This is another key metric for optimization. It is worth converting buffer operation numbers to bytes. In most cases, buffer size is 8 KiB (check: show block_size;), so `500,000` buffer hits&reads per second translates to `500000 bytes/sec * 8 / 1024 / 1024 =  ~ 3.8 GiB/s` of the internal data reading flow (again: the same buffer in the pool can be process multiple times). This is a significant load – you might want to check the other metrics to understand if it is reasonable to have or it is a candidate for optimization.

* `dM/dt`, where `M` is `wal_bytes` – the stream of WAL bytes written. This is relatively new metric (PG13+) and can be used to understand which queries contribute to WAL writes the most – of course, the more WAL is written, the higher pressure to physical and logical replication, and to the backup systems we have. An example of highly pathological workload here is: a series of transactions like `begin; delete from ...; rollback;` deleting many rows and reverting this action – this produces a lot of WAL not performing any useful work. // TODO: failing queries are not tracked in pgss, so this example is not ideal – find a better one

=== 

That's it for the part 1 of pgss-related howto, in next parts we'll talk about dM/dc and %M, and other practical aspects of pgss-based macro optimization.

Let me know if it was useful, and please share with your colleagues and any people who work with 
PostgreSQL.
.