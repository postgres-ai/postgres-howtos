Originally from: [tweet](https://twitter.com/samokhvalov/status/1740813478150189172), [LinkedIn post]().

---

# How to tune work_mem

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

`work_mem` is used for sorting and hashing operations during query execution.

By default it's 4MB.

Due to its nature, the tuning of `work_mem` is quite tricky.

One of the possible approaches is explained here.

## Rough tuning and "safe" values of work_mem

First, apply rough optimization as described in
[Day 89: Rough configuration tuning (80/20 rule; OLTP)](0089_rough_oltp_configuration_tuning.md).

A query can "spend" `work_mem` multiple times (for multiple operations). But it is not allocated fully for
each operation – an operation can need a lower amount of memory.

Therefore, it is hard to reliably predict, how much memory we'll need to use to handle a workload, without actual
observation of the workload.

Moreover, in Postgres 13, new parameter was added,
[hash_mem_multiplier](https://postgresqlco.nf/doc/en/param/hash_mem_multiplier/), adjusting the logic. The default is 2
in PG13-16. It means that max memory used by a single hash operation is 2 * `work_mem`.

Worth mentioning, understanding how much memory is used by a session in Linux is very tricky per se – see a great
article by Andres Freund:
[Analyzing the Limits of Connection Scalability in Postgres](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/analyzing-the-limits-of-connection-scalability-in-postgres/ba-p/1757266#memory-usage) 
(2020).

A safe approach would be:

- estimate how much memory is free – subtracting `shared_buffers`, `maintenance_work_mem`, etc.,
- then divide the estimated available memory by `max_connections` and additional number such as 4-5 (or more, to be on
  even safer side) – assuming that each backend will be using up to 4*`work_mem` or 5*`work_mem`. Of course, this multiplier
  itself is a very rough estimate – in reality, OLTP workloads usually are much less hungry on average (e.g., having a
  lot of PK lookups mean that average memory consumption is very low).

In practice, it can make sense to adjust `work_mem` to a higher value, but this needs to be done after understanding the
behavior of Postgres under certain workload. The following steps are parts of iterative approach for further tuning.

## Temp files monitoring

Monitor how often temporary files are created and the size of these files. Source of proper info:

1) `pg_stat_database`, columns `temp_files` and `temp_bytes`.

2) Postgres logs – set `log_temp_files` to a low value, down to 0 (being careful with the observer effect).

3) `temp_blks_read` and `temp_blks_written` in `pg_stat_statements`.

The goal of further `work_mem` tuning is getting rid of temporary files completely, or minimizing the frequency of
creation of them and the size.

## Optimize queries

If possible, consider optimizing queries that involve temporary files creation. Spend a reasonable amount of effort on
this - if there is no obvious optimization, proceed to the next step.

## Raising work_mem further: how much?

If reasonable optimization efforts are done, it is time to consider raising `work_mem` further.

However, how much? The answer lies in the size of individual temporary files – from monitoring described above, we can
find maximum and average temp file size, and estimate the required raise from there. 

> 🎯 **TODO:** detailed steps

## Raise work_mem for parts of workload

It makes sense to raise it for individual queries. Consider two options:

- `set work_mem ...` in individual sessions or transactions (`set local ...`), or
- if you use various DB users for various parts of your workload, consider adjusting `work_mem` for particular users 
  (e.g., for those that run analytical-like queries): `alter user ... set work_mem ...`.

## Raise work_mem globally

Only if the previous steps are not suitable (e.g., it is hard to optimize queries and you cannot tune `work_mem` for parts
of workload), then consider raising `work_mem` globally, evaluating OOM risks.

## Iterate

After some time, review data from monitoring to ensure that situation improved or decide to perform another iteration.

## Extra: pg_get_backend_memory_contexts

In Postgres 14, a new function `pg_get_backend_memory_contexts()` and corresponding view were
added; see [docs](https://postgresql.org/docs/current/view-pg-backend-memory-contexts.html). This is helpful for detailed analysis
of how current session works with memory, but the major limitation of this capability is that it works only with the
current session.

> 🎯 **TODO:** How to use it.
