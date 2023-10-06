Originally from: [tweet](https://twitter.com/samokhvalov/status/1709069225762095258), [LinkedIn post](https://www.linkedin.com/pulse/how-work-pgstatstatements-part-3-nikolay-samokhvalov/). 

---

# How to work with pg_stat_statements, part 3
Previous parts:
- [part 1](./0005_pg_stat_statements_part_1.md)
- [part 2](./0006_pg_stat_statements_part_2.md)

## 3rd type of derived metrics: percentage
Now, let's examine the third type of derived metrics: the percentage that a considered query group (normalized query or bigger groups such as "all statements from particular user" or "all `UPDATE` statements") takes in the whole workload with respect to metric `M`.

How to calculate it: first, apply time-based differentiation to all considered groups (as discussed in [the part 1](././0005_pg_stat_statements_part_1.md)) — `dM/dt` — and then divide the value for particular group by the sum of values for all groups.

## Visualization and interpretation of %M

While `dM/dt` gives us absolute values such as calls/sec or `GiB/sec`, the `%M` values are relative metrics. These values help us identify the "major players" in our workload considering various aspects of it — frequency, timing, IO operations, and so forth.

Analysis of relative values helps understand how big is the potential win from each optimization vector and prioritize our optimization activities, first focusing on those having the most potential. For example:
- If the absolute value on QPS seems to be high — say, `1000 calls/sec` — but if it represents just `3%` of the whole workload, an attempt to reduce this query won't give a big win, and if we are concerned about QPS, we need to optimize other query groups.
- However,  if we have `1000 calls/sec` and see that it's `50%` of the whole, this single optimization step — say, reducing it to `10 calls/sec` — helps us shave off almost half of all the QPS we have.

One of the ways to deal with proportion values in larger systems is to react on large percentage values, consider the corresponding query groups as candidates for optimization. For example, in systems with large number of query groups, it might make sense to apply the following approach:
- Periodically, for certain metrics (for example, `calls`, `total_exec_time`, `total_plan_time`, `shared_blks_dirtied`, `wal_bytes`), build Top-10 lists showing query groups having the largest `%M` values. 
- If particular query group turns out to be a major contributor – say, >20% — on certain metrics, consider this query as a candidate for optimization. For example, in most cases, we don't want a single query group to be responsible for 1/2 of the aggregated `total_exec_time` ("total `total_exec_time`", apologies for tautology).
– In certain cases, it is ok to decide that query doesn't require optimization — in this case we mark such group as exclusion and skip it in the next analyses.

The analysis of proportions can also be performed implicitly, visually in monitoring system: observing graphs of `dM/dt` (e.g., QPS, block hits per second), we can visually understand which query group contributes the most in the whole workload, considering a particular metric `M`. However, for this, graphs need to be ["stacked"](https://en.wikipedia.org/wiki/Bar_chart#Grouped_.28clustered.29_and_stacked).

If we deal with 2 snapshots, then it makes sense to obtain such values explicitly. Additionally, for visualization purposes, it makes sense to draw a pie chart for each metric we are analyzing.

## %M examples
1. `%M`, where `M` is `calls` — this gives us proportions of QPS. For example, if we normally have `~10k QPS`, but if some query group is responsible for `~7k QPS`, this might be considered as abnormal, requiring optimizations on client side (usually, application code).

2. `%M`, where `M` is `total_plan_time + total_exec_time` — percentage in time that the server spends to process queries in a particular group. For example, if the absolute value is `20 seconds/second` (quite a loaded system — each second Postgres needs to spend 20 seconds to process queries), and a particular group has 75% on this metric, it means we need to focus on optimizing this particular query group. Ways to optimize:
    - If QPS (`calls/second`) is high, then, first of all, we need to reduce. 
    - If average latency (`total_exec_time`, less often `total_plan_time` or both) is high, then we need to apply micro-optimization using `EXPLAIN` and `EXPLAIN (ANALYZE, BUFFERS)`.
    - In some cases, we need to combine both directions of optimization.

3. `%M`, where `M` is `shared_blks_dirtied` — percentage of changes in the buffer pool performed by the considered query group. This analysis may help us identify the write-intensive parts of the workload and find opportunities to reduce the volume of checkpoints and amount of disk IO.

4. `%M`, where `M` is `wal_bytes` — percentage of bytes written to WAL. This helps us identify those query groups where optimization will be more impactful in reducing the WAL volumes being produced.

## Instead of summary: three macro-optimization goals and what to use for them
Now, with the analysis methods described here and in the previous 2 parts, let's consider three popular types of macro-optimization with respect to just a single metric — `total_exec_time`.

Understanding each of these three approaches (and then applying this logic to other metrics as well) can help you understand how your monitoring dashboards should appear.

1. Macro-optimization aimed to reduce resource consumption. Here we want to reduce, first of all, CPU utilization, memory and disk IO operations. For that, we need to use the `dM/dt` type of derived metric – the number of seconds Postgres spends each second to process the queries. Reducing this metric — the aggregated "total" value of it for the whole server, and for the Top-N groups playing the biggest role — has to be our goal. Additionally, we may want to consider other metrics such as `shared_blks_***`, but the timing is probably the best starting point. This kind of optimization can help us with capacity planning, infrastructure cost optimization, reducing risks of resource saturation.

2. Macro-optimization aimed to improve user experience. Here we want our users to have the best experience, therefore, in the OLTP context, we should focus on average latencies with a goal to reduce them. Therefore, we are going to use `dM/dc` — number of seconds (or milliseconds) each query lasts on average. If in the previous type of optimization, we would like to see Top-N query groups ordered by the `dM/dt` values (measured in `seconds/second`) in our monitoring system, here we want Top-N ordered by avg. latency (measured in seconds). Usually, this gives us a very different set of queries — perhaps, having lower QPS. But these very queries have worst latencies and these very queries annoy our users the most. In some cases, in this this type of analysis, we might want to exclude query groups with low QPS (e.g., those having QPS < 1 call/sec) or exclude specific parts of workload such as data dump activities that have inevitably long latencies.

3. Macro-optimization aimed to balance workload. This is less common type of optimization, but this is exactly where `%M` plays its role. Developing our application, we might want, from time to time, check Top-N percentage on `total_exec_time + total_plan_time` and identify query groups holding the biggest portion — exactly as we discussed above.

Bonus: podcast episodes

Related [Postgres.FM](https://postgres.fm) episodes::
- [Intro to query optimization](https://postgres.fm/episodes/intro-to-query-optimization)
- [102 query optimization](https://postgres.fm/episodes/102-query-optimization)
- [pg_stat_statements](https://postgres.fm/episodes/pg_stat_statements)

---

That concludes our discussion on pgss for now. Of course, this was not a complete guide, we might return to this important extension in the future.

[Let me know](https://twitter.com/samokhvalov) if you have questions or any other feedback!
