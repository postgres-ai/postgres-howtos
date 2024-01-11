Originally from: [tweet](https://twitter.com/samokhvalov/status/1714543861975204241), [LinkedIn post]().

---

# How to analyze heavyweight locks, part 1

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Heavyweight locks, both relation- and row-level, are acquired by a query and always held until the end of the
transaction this query belongs to. So, important principle to remember: once acquired, a lock is not released until
`COMMIT` or `ROLLBACK`.

Docs: [Explicit locking](https://postgresql.org/docs/current/explicit-locking.html). A few notes about this doc:

- The title "Explicit Locking" might seem misleading – it actually describes the levels of locks that can be acquired
  implicitly by any statement, not just explicitly via `LOCK`.
- This page also contains a very useful table, "Conflicting Lock Modes", that helps understand the rules according which
  certain locks cannot be acquired due to conflicts and need to wait until the transaction holding such locks finishes,
  releasing the "blocking" locks. This article has an alternative table that might be also helpful:
  [PostgreSQL rocks, except when it blocks: Understanding locks](https://citusdata.com/blog/2018/02/15/when-postgresql-blocks/)
- There also might be confusion in terminology. When discussing "table-level" locks, we might actually mean
  "relation-level" locks. Here, the term "relation" assumes a broader meaning: tables, indexes, views, materialized
  views.

How can we see which locks have already been acquired (granted), or are being attempted but not yet acquired (pending)
for a particular transaction/session?

For this, there is a system view: [pg_locks](https://postgresql.org/docs/current/view-pg-locks.html).

Important rule to remember: the analysis should be conducted in a separate session, to exclude the "observer effect"
(the locks that are acquired by the analysis itself).

For example, consider a table:

```
nik=# \d t1
                Table "public.t1"
 Column |  Type  | Collation | Nullable | Default
--------+--------+-----------+----------+---------
 c1     | bigint |           |          |
Indexes:
    "t1_c1_idx" btree (c1)
    "t1_c1_idx1" btree (c1)
    "t1_c1_idx10" btree (c1)
    "t1_c1_idx11" btree (c1)
    "t1_c1_idx12" btree (c1)
    "t1_c1_idx13" btree (c1)
    "t1_c1_idx14" btree (c1)
    "t1_c1_idx15" btree (c1)
    "t1_c1_idx16" btree (c1)
    "t1_c1_idx17" btree (c1)
    "t1_c1_idx18" btree (c1)
    "t1_c1_idx19" btree (c1)
    "t1_c1_idx2" btree (c1)
    "t1_c1_idx20" btree (c1)
    "t1_c1_idx3" btree (c1)
    "t1_c1_idx4" btree (c1)
    "t1_c1_idx5" btree (c1)
    "t1_c1_idx6" btree (c1)
    "t1_c1_idx7" btree (c1)
    "t1_c1_idx8" btree (c1)
    "t1_c1_idx9" btree (c1)
```

In the first (main) session:

```
nik=# begin;
BEGIN

nik=*# select from t1 limit 0;
--
(0 rows)
```

– we opened a transaction, performed a `SELECT from t1` - requesting 0 rows and 0 columns, but this is enough to acquire
relation-level locks. To view these locks, we need to first obtain the `PID` of the first session, running this inside
it:

```
nik=*# select pg_backend_pid();
 pg_backend_pid
----------------
          73053
(1 row)
```

Then, in a separate session:

```
nik=# select relation, relation::regclass as relname, mode, granted, fastpath
from pg_locks
where pid = 73053 and locktype = 'relation'
order by relname;
 relation |   relname   |      mode       | granted | fastpath
----------+-------------+-----------------+---------+----------
    74298 | t1          | AccessShareLock | t       | t
    74301 | t1_c1_idx   | AccessShareLock | t       | t
    74318 | t1_c1_idx1  | AccessShareLock | t       | t
    74319 | t1_c1_idx2  | AccessShareLock | t       | t
    74320 | t1_c1_idx3  | AccessShareLock | t       | t
    74321 | t1_c1_idx4  | AccessShareLock | t       | t
    74322 | t1_c1_idx5  | AccessShareLock | t       | t
    74323 | t1_c1_idx6  | AccessShareLock | t       | t
    74324 | t1_c1_idx7  | AccessShareLock | t       | t
    74325 | t1_c1_idx8  | AccessShareLock | t       | t
    74326 | t1_c1_idx9  | AccessShareLock | t       | t
    74327 | t1_c1_idx10 | AccessShareLock | t       | t
    74328 | t1_c1_idx11 | AccessShareLock | t       | t
    74337 | t1_c1_idx12 | AccessShareLock | t       | t
    74338 | t1_c1_idx13 | AccessShareLock | t       | t
    74339 | t1_c1_idx14 | AccessShareLock | t       | t
    74345 | t1_c1_idx20 | AccessShareLock | t       | f
    74346 | t1_c1_idx15 | AccessShareLock | t       | f
    74347 | t1_c1_idx16 | AccessShareLock | t       | f
    74348 | t1_c1_idx17 | AccessShareLock | t       | f
    74349 | t1_c1_idx18 | AccessShareLock | t       | f
    74350 | t1_c1_idx19 | AccessShareLock | t       | f
(22 rows)
```

Notes:

- For brevity, we are only requesting locks with the `locktype` set to `'relation'`. In a general case, we might be
  interested in other lock types too.
- To see relation names, we convert `oid` values to `regclass`, this is the shortest way to retrieve the table/index
  names (so, `select 74298::oid::regclass` returns `t1`).
- The lock mode `AccessShareLock` is the "weakest" possible, it blocks operations like `DROP TABLE`, `REINDEX`, certain
  types of `ALTER TABLE/INDEX`. By locking both the table and all its indexes, Postgres guarantees that they will remain
  present during our transaction.
- Again: **all** indexes are locked with `AccessShareLock`.
- In this case, all locks are granted. One might think it is always so with `AccessShareLock`, but it's not – if there
  is a granted or **pending** `AccessExclusiveLock` (the "strongest" on), then our attempt to acquire
  an `AccessShareLock` will be in the pending state. When might a pending `AccessExclusiveLock` occur? If there is an
  attempt of `AccessExclusiveLock` (e.g. `ALTER TABLE`), but there is some long-lasting `AccessShareLock` – a "sandwich"
  situation. This scenario can lead to downtimes when, during an attempt to deploy a very
  simple `ALTER TABLE .. ADD COLUMN` without proper precaution measures (low `lock_timeout` and retries), it is blocked
  by a long-running `SELECT`, which, in its turn, blocks subsequent `SELECT`s (and other DML). More:
  [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries).
- Only 16 of the locks have `fastpath=true`. When `fastpath=false`, Postgres lock manager uses a slower, but more
  comprehensive method to acquire locks. It is discussed in #PostgresMarathon
  [Day 18: Over-indexing](0018_over_indexing.md).
