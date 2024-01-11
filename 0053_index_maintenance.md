Originally from: [tweet](https://twitter.com/samokhvalov/status/1725784211809059101), [LinkedIn post]().

---

# Index maintenance

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Index maintenance is inevitable in a larger project. The sooner the process is established the better for performance.

## Analyze index health

Index health analysis includes:

- Identifying invalid indexes
- Bloat analysis
- Finding unused indexes
- Finding redundant indexes
- Checking for corruption

## Invalid indexes

Finding invalid indexes is simple:

```
nik=# select indexrelid, indexrelid::regclass, indrelid::regclass
from pg_index
where not indisvalid;
 indexrelid | indexrelid | indrelid
------------+------------+----------
      49193 | t_id_idx   | t1
(1 row)
```

A bit more comprehensive query can be found in [Postgres DBA](https://github.com/NikolayS/postgres_dba/).

When analyzing this list, keep in mind that an invalid index may be a normal situation if this is an index that is being
built or rebuild by `CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY`, so it is worth also checking
`pg_stat_activity` to identify such processes.

The other invalid indexes have to be rebuilt (`REINDEX CONCURRENTLY`) or dropped (`DROP INDEX CONCURRENTLY`).

## Bloated indexes

How to analyze index bloat: see [Day 46: How to deal with bloat](./0046_how_to_deal_with_bloat.md).

Indexes with high bloat (real or estimated) – say, above 50% – have to be reindexed (`REINDEX CONCURRENTLY`).

Reindexing is needed because over time, due to updates index health degrades. Certain things can help to slow down this
degradation:

- Upgrading to Postgres 14 or newer (to benefit from btree optimizations).
- Optimize workload and schema to make
  more `UPDATES HOT` ([Heap Only Tuple](https://postgresql.org/docs/current/storage-hot.html)).
- Tune `autovacuum` for more active vacuuming (though, remember that `VACUUM` doesn't rebalance btree indexes).

Thus, from time to time, you need to use `REINDEX CONCURRENTLY` for those indexes that are known to be bloated. Ideally,
this procedure needs to be automated. Example: [pg_auto_reindexer](https://github.com/vitabaks/pg_auto_reindexer).

## Unused indexes

Usage information can be found in [pg_stat_user_indexes](https://postgresql.org/docs/current/monitoring-stats.html).
An example of query for analysis can be found in [Postgres DBA](https://github.com/NikolayS/postgres_dba/).

When searching for unused indexes, be careful and avoid mistakes:

1) Ensure that stats were reset long enough ago. For example, if it happened only a couple of days ago, and you think an
   index is unused, it might be a mistake – if this index will be needed to support some reports on the 1st day of the
   next month.
2) Don't forget to analyze all the nodes that receive workload – the primary and all replicas. An index that looks
   unused on the primary may be needed on a replica.
3) If you have multiple installation of your system, make sure you analyzed all of them or at least representative
   portion of them.

Once unused indexes are identified reliably, they need to be dropped using `DROP INDEX CONCURRENTLY`.

Can we soft-drop index ("hide" it from the planner to ensure that planner behavior doesn't change and if so, proceed
with real dropping, otherwise quickly reverting to the original state)? There is no simple answer here, unfortunately:

1) [HypoPG 1.4.0](https://github.com/HypoPG/hypopg/releases/tag/1.4.0) has a feature to "hide" indexes – this is very
   useful, but you need to install it and, more importantly, and it might be challenging to use it for whole workload,
   since you need to call `hypopg_hide_index(oid)` for it.
2) Some people use a trick with setting `indisvalid` to `false` to hide an index from the planner – but there is a
   reliable opinion that this is a not safe approach; see
   [Peter Geoghegan's Tweet](https://twitter.com/petervgeoghegan/status/1599191964045672449):

   > It's unsafe, basically. Though hard to say just how likely it is to break. Here is one hazard that I know of: in
   > general such an update might break a concurrent `pg_index.indcheckxmin = true` check. It will effectively
   > "change the xmin" of your affected row, confusing the check.

## Redundant indexes

An index A supporting a set of queries is redundant to index B if B can efficiently support the same set of queries and,
optionally, more. A few examples:

- Index on column (a) is redundant to index on (a, b).
- Index on column (a) is NOT redundant to index on (b, a).

Indexes having exactly the same definition are redundant to each other (a.k.a. duplicate indexes).

An example of a query to identify redundant indexes can be found in
[Postgres DBA](https://github.com/NikolayS/postgres_dba/)
or [Postgres Checkup](https://gitlab.com/postgres-ai/postgres-checkup).

Redundant indexes are usually safe to drop (after double-checking the list of them manually), even if the index which is
going to be dropped, is currently used.

## Corruption

Use `pg_amcheck` to identify corruption in btree indexes (TODO: details – will be in a separate howto).

As of 2023 / PG16, the following features are not yet supported by `pg_amcheck`, but there are plans to add them in the
future:

- [Checking GIN and GIST](https://commitfest.postgresql.org/45/3733/)
- Checking unique keys (already pushed, potentially to be released with PG17)
