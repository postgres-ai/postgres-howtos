Originally from: [tweet](https://twitter.com/samokhvalov/status/1711575029006418218), [LinkedIn post](...).

---

# How to decide when a query is too slow and needs optimization

<img src="files/0014_cover.png" width="600" />

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

"Slow" is a relative concept. In some cases, we might be happy with query latency 1 minute (or no?), while in other
scenarios, even 1 ms might seem to be too slow.

Decision when to apply optimization techniques is important for efficiency – as Donald Knuth famously stated in "The Art
of Computer Programming":

> The real problem is that programmers have spent far too much time worrying about efficiency in the wrong places and at
> the wrong times; premature optimization is the root of all evil (or at least most of it) in programming.

Below we assume that we work with OLTP or hybrid workloads and need to decide if a certain query is too slow and
requires optimization.

## How to conclude that a query is too slow

1. Do you have an OLTP case or an analytical one, or hybrid? For OLTP cases, requirements are more strict and dictated
   by human perception (see: [What is a slow SQL query?](https://postgres.ai/blog/20210909-what-is-a-slow-sql-query)),
   while for analytical needs, we can usually wait a minute or two – unless it's also user-facing. If it is, we probably
   consider 1 minute as too slow. In this case, consider using column-store database systems (and Postgres ecosystem has
   a new offering here: check out [@hydradatabase](https://twitter.com/hydradatabase)). For OLTP, the majority of
   user-facing queries should be below 100ms – ideally, below 10ms – so the complete
   requests to your backends that users make, do not exceed 100-200ms (each request can issue several SQL queries,
   depending on the case). Of course, non-user-facing queries such as those coming from background jobs, `pg_dump`, and so
   on, can last longer – assuming that the next principles are met.

2. In the case of OLTP, the second question should be: is this query "read-only" or it changes the data (be it DDL or
   just writing DML – INSERT/UPDATE/DELETE)? In this case, in OLTP, we shouldn't allow it to run longer than a second or
   two, unless we are 100% sure that this query won't block other queries for long. For massive writes, consider
   splitting them in batches so each batch doesn't last longer than 1-2 seconds. For DDL, be careful with lock
   acquisition and lock chains (read these
   posts: [Common DB schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes#case-5-acquire-an-exclusive-lock--wait-in-transaction)
   and
   [Useful queries to analyze PostgreSQL lock trees (a.k.a. lock queues)](https://postgres.ai/blog/20211018-postgresql-lock-trees)).

3. If you're dealing with a read-only query, make sure it's also not running for too long – long-running transactions
   make Postgres hold old dead tuples for long ("xmin horizon" is not advancing), so autovacuum cannot delete dead
   tuples that became dead after the start of our transaction. Aim to avoid transactions that last longer than one or a
   few hours (and if you absolutely need such long transactions, prefer running them at low-activity hours, when XID is
   progressing slowly, and do not run them often).

4. Finally, even if a query is relatively fast – for instance, 10ms – it might still be considered too slow if its
   frequency is high. For example, 10ms query running 1,000 times per second (you can check it via
   `pg_stat_statements.calls`), then Postgres needs to spend 10 seconds *every* second to process this group of queries.
   In this case, if lowering down the frequency is hard, the query should be considered slow, and an optimization
   attempt needs to be performed, to reduce resource consumption (the goal here is to reduce
   `pg_stat_statements.total_exec_time` – see
   the [previous #PostgresMarathon posts about pgss](https://twitter.com/search?q=%23PostgresMarathon%20pg_stat_statements&src=typed_query&f=live)).

## Summary

- All queries that last longer than 100-200 ms should be considered as slow, if they are user-facing. Good queries are
  those that are below 10 ms.
- Background processing queries are ok to last longer. If they modify data and might block user-facing queries, then
  they should not be allowed to last longer than 1-2 s.
- Be careful with DDLs – make sure they don't cause massive writes (if they do, it should be split into batches as
  well), and use low `lock_timeout` and retries to avoid blocking chains.
- Do not allow long-running transactions. Make sure the xmin horizon is progressing and autovacuum can remove dead
  tuples promptly – do not allow transactions that last too long (>1-2h).
- Optimize even fast (<100ms) queries if the corresponding `pg_stat_statements.calls` and
  `pg_stat_statements.total_exec_time` are high.
