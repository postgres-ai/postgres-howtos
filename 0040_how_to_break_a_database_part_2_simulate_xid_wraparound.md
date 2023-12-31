Originally from: [tweet](https://twitter.com/samokhvalov/status/1721036531580973494), [LinkedIn post]().

---

# How to break a database, Part 2: Simulate infamous transaction ID wraparound

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

See also [Part 1: How to Corrupt](0039_how_to_break_a_database_part_1_how_to_corrupt.md).

## Straightforward simulation method

The method requires some time but works pretty well. It has been discussed in a couple of places:

- My tweet [A question from Telegram chat](https://twitter.com/samokhvalov/status/1415575072081809409)
- Article
  [How to simulate the deadly, infamous, misunderstood & complex ‘Transaction Wraparound Problem’ in PostgreSQL](https://fatdba.com/2021/07/20/how-to-simulate-the-deadly-transaction-wraparound-problem-in-postgresql/)

First, open a "long-running transaction" (with normal XID assigned to it, as it would be a writing transaction)  – and
keep the session open (we'll use a named pipe a.k.a. FIFO to "hang" our `psql` session):

```bash
mkfifo dummy

psql -Xc "
  set idle_in_transaction_session_timeout = 0;
  begin;
  select pg_current_xact_id()
" \
-f dummy &
```

Make sure the session is open and in transaction:

```bash
❯ psql -Xc "select state
  from pg_stat_activity
  where
    pid <> pg_backend_pid()
    and query ~ 'idle_in_tran'
"
        state
---------------------
 idle in transaction
(1 row)
```

Now, having an open long-running transaction with assigned XID, we just need to shift transaction ID at high speed – we
can use the same function for it, and `pgbench` with multiple connections:

```bash
pgbench -c8 -j8 -P60 -T36000 -rn \
  -f - <<< 'select pg_current_xact_id()'
```

This should move the current XID at very high pace (100-200k TPS). And since we have an open a long-running
transaction, `autovacuum` cannot operate.

While `pgbench` is running, we can observe the state of the database using a monitoring tool that includes XID/multiXID
wraparound checks (every tool must, but not every tool does), or just some snippet – e.g.
from [Managing Transaction ID Exhaustion (Wraparound) in PostgreSQL](https://crunchydata.com/blog/managing-transaction-id-wraparound-in-postgresql).

After a few hours:

```
WARNING:  database "nik" must be vacuumed within 39960308 transactions
HINT:  To avoid a database shutdown, execute a database-wide VACUUM in that database.
You might also need to commit or roll back old prepared transactions, or drop stale replication slots.
```

And if we proceed, then this:

```bash
nik=# create table t3();
ERROR:  database is not accepting commands to avoid wraparound data loss in database "nik"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
You might also need to commit or roll back old prepared transactions, or drop stale replication slots.
```

Good, the harm is done! It's time to release FIFO (this will close our long-lasting idle-in-transaction session):

```bash
exec 3>dummy && exec 3>&-
```

## How to escape – the basics

How to escape? We'll probably discuss this in a separate howto, but let's talk about some basics.

See real-life use cases to learn from the other people's mistakes:

- [Sentry's case](https://blog.sentry.io/transaction-id-wraparound-in-postgres/),
- [Mailchimp's one](https://mailchimp.com/what-we-learned-from-the-recent-mandrill-outage/)

are good lessons.

Escaping from this takes, in general case, takes a lot of time. Traditionally, as the `HINT` above suggests, it is done
in the single-user mode. But I very much like the ideas by
[@PostSQL](https://twitter.com/PostSQL)
from the GCP team – see the talk [Do you vacuum every day?](https://youtube.com/watch?v=JcRi8Z7rkPg) and discussion in
the `pgsql-hackers` mailing list
[We should stop telling users to "vacuum that database in single-user mode"](https://postgresql.org/message-id/flat/CAMT0RQTmRj_Egtmre6fbiMA9E2hM3BsLULiV8W00stwa3URvzA%40mail.gmail.com).

## Alternative method to simulate wraparound

There is another method for simulation which works much faster – no need to wait many hours, we can use `pg_resetwal`'s
option `-x` ([docs](https://postgresql.org/docs/current/app-pgresetwal.html)) to fast-forward the transaction ID. This
method is described in [Transaction ID wraparound: a walk on the wild
side](https://cybertec-postgresql.com/en/transaction-id-wraparound-a-walk-on-the-wild-side/). It's quite interesting
and looks like this:

```bash
pg_ctl stop -D $PGDATA

pg_resetwal -x 2137483648 -D testdb ### 2^31 - 10000000

dd if=/dev/zero of=$PGDATA/pg_xact/07F6 bs=8192 count=15
```

## MultiXact ID wraparound

MultiXact ID is also possible:
[Multixacts and Wraparound](https://postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-MULTIXACT-WRAPAROUND).
This risk is often overlooked. But it exists, especially if you have FKs and/or use
`SELECT ... FOR SHARE` (and `SELECT ... FOR UPDATE`) combined with substransactions).

Some articles to learn about it:

- [A foreign key pathology to avoid](https://thebuild.com/blog/2023/01/18/a-foreign-key-pathology-to-avoid/)
- [Subtransactions considered harmful: Problem 3: unexpected use of Multixact IDs](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful#problem-3-unexpected-use-of-multixact-ids)
- [Notes on some PostgreSQL implementation details](https://buttondown.email/nelhage/archive/notes-on-some-postgresql-implementation-details/)

Simulation of MultiXact ID wraparound is also possible – for this, we need to create multiple overlapping transactions
that lock the same rows (for example, using explicit `SELECT ... FOR SHARE`).

> 🎯 **TODO:** working example – left for future edits.

Everyone should not only monitor (with alerting) for traditional XID wraparound risks, but also for MultiXID wraparound
risks. And it should be included in the snippets showing how much of the "capacity" is used.

> 🎯 **TODO:** snippet to show both XID w-d and MultiXID w-d, at both DB and table level
> Obviously, multixid wraparounds are encountered less often in the wild – I don't see people have `datminmxid` and
> `relminmxid` used in the snippets.
> Basic version:
>
> ```sql
> select 
>   datname, 
>   age(datfrozenxid), 
>   mxid_age(datminmxid)
> from pg_database;
> ```
