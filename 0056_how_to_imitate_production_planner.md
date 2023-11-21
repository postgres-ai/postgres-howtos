Originally from: [tweet](https://twitter.com/samokhvalov/status/1726887992340738324), [LinkedIn post]().

---

# How to make the non-production Postgres planner behave like in production

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

For query optimization goals – running `EXPLAIN (ANALYZE, BUFFERS)` and verifying various optimization ideas such as a
new index – it is crucial to ensure that the Postgres planner behaves exactly like or similarly to the planner working
with the production database. Fortunately, this is possible regardless of the resources (CPU, RAM, disk IO) available on
the non-production machine.

The technique described here is used in DBLab Engine
[@Database_Lab](https://twitter.com/Database_Lab)
to support fast database cloning/branching for query optimization and database testing in CI/CD.

To achieve the planner's prod/non-prod behavior parity, two components are needed:

1) Matching database settings
2) The same or very similar statistic (the content of `pg_statistic`)

## Matching database settings

Use this query on the production database to obtain the list of non-default settings that influence the planner's
behavior:

```sql
select
  format(e'%s = %s', name, setting) as configs
from
  pg_settings
where
  source <> 'default'
  and (
    name ~ '(work_mem$|^enable_|_cost$|scan_size$|effective_cache_size|^jit)'
    or name ~ '(^geqo|default_statistics_target|constraint_exclusion|cursor_tuple_fraction)'
    or name ~ '(collapse_limit$|parallel|plan_cache_mode)'
  );
```

Notes:

- The planner's behavior doesn't depend on the actual resources available such as CPU or RAM. Nor does it depend on OS,
  FS or their settings.
- The value of `shared_buffers` doesn't matter(!) – it will only affect the executor's behavior and buffer pool's
  hit/read ratio. What does matter for the planner is `effective_cache_size` and you can set it to a value that
  significantly exceed the actual RAM available, "fooling" the planner in a good sense, achieving the goal to match the
  production planner behavior. So you can have, say, 1 TiB of RAM and `shared_buffers = '250GB'` in production and
  `effective_cache_size = '750GB'`, and be able to effectively analyze and optimize queries on a small 8-GiB machine
  with `shared_buffers = '2GB'` and `effective_cache_size = '750GB'`. The planner will assume you have a lot of RAM when
  choosing the optimal plans.
- Similarly, you can have fast SSDs in production and use `random_page_cost = 1.1` there, and very slow cheap magnetic
  disks in your testing environments. With `random_page_cost = 1.1` being used there, the planner will consider random
  access to be not costly. The execution will be still slow, but the plan chosen and the data volumes (actual rows,
  buffer numbers) will be the same or very close to production.

## Matching statistics

There is a quite interesting and exotic idea of dumping/restoring the content of `pg_statistic` from production to testing
environments of much smaller size, not having the actual data there. There is not a very popular solution implementing
this idea; possibly, [pg_dbms_stats](https://github.com/ossc-db/pg_dbms_stats/blob/master/doc/pg_dbms_stats-en.md) can
help with it.

> 🎯 **TODO:** Test and see if it works.

A straightforward idea is to have the same-size database. Two options are possible here:

* **Option 1:** physical copy. It can be done using `pg_basebackup`, copying `PGDATA` using rsync or an alternative, or
  restoring from physical backups such as `WAL-G`'s or `pgBackRest`'s.

* **Option 2:** logical copy. It can be done using dump/restore or creating a logical replica.

Notes:

- The physical method of provisioning gives the best results: not only will "actual rows" match, but also the buffer
  numbers provided by running `EXPLAIN (ANALYZE, BUFFERS)`, because the number of pages (`relpages`) is the same, the
  bloat
  is preserved, and so on. This is the best method for optimization and fine-tuning the queries.
- The logical method gives the matching row counts, but the size of tables and indexes is going to be
  different – `relpages` is smaller in a freshly provisioned node, the bloat is not preserved, and tuples, generally,
  are stored in a different order (we can call this bloat "good" since we want to have it in testing environments to
  match the production state). This method still enables quite efficient query optimization workflow, with an additional
  idea that the importance of keeping bloat low in production becomes higher.
- After dump/restore you must explicitly run `ANALYZE` (or `vacuumdb --analyze -j <number of workers>`) to initially
  gather statistics in `pg_statistic`, because `pg_restore` (or `psql`) won't run it for you.
- If the database content needs to be changed to remove sensitive data, this most certainly is going to affect the
  planner behavior. For some queries, the impact may be quite low, but for others it can be critical, making query
  optimization virtually impossible. These negative effects are generally grater than those caused by dump/restore
  losing bloat because:
    - dump/restore affects `relpages` (bloat lost) and the order of tuples, but not the content of `pg_statistic`
    - removal of sensitive data can not only reorder tuples, produce irrelevant bloat ("bad bloat"), but also lose
      important parts of production's `pg_statistic`.

## Summary

To make the non-production Postgres planner behave like in production, perform these two steps:

1) Tune certain non-prod Postgres settings (planner-related, `work_mem`) to match production.

2) Copy the database from production if possible, preferring physical type of provisioning to logical, and – if it is
   possible, of course – avoiding data modifications. For ultra-fast delivery of clones, consider using the DBLab Engine
   [@Database_Lab](https://twitter.com/Database_Lab).
