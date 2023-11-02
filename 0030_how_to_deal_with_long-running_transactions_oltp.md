Originally from: [tweet](https://twitter.com/samokhvalov/status/1717429427808911813), [LinkedIn post]().

---

## How to deal with long-running transactions (OLTP)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## Why long-running transactions can be a problem

In the OLTP context (e.g., mobile and web apps), long-running transactions are often harmful for two reasons:

1. Blocking issues. Locks, once acquired, are released only at the end of the transaction. This can block other
   transactions. And sometimes, even the "weakest" possible lock – `AccessShareLock` – can be a big problem when being
   held for too long;
   [even a simple open transaction that has read from a table, can cause big troubles](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries#problem-demonstration).

2. Negative effects to autovacuum activities. If we have an open transaction with some transaction ID – say, `xid1`, all
   dead tuples with `xmax > xid1` (in other words, tuples that became dead after our transaction has started - those
   tuples that are produced by some transaction with `XID > xid1`) cannot be deleted by autovacuum until our transaction
   finishes. This might lead to bloat and performance degradation.

"Long-running" is a relative term, and, of course, its meaning depends on particular situation. Usually, in
heavily-loaded systems – say ~10^5 TPS including RO queries and ~10^3 of XID-consuming TPS (writes) – we consider
transactions running longer than 30-60 seconds to be long. This can be translated to 30-60k dead tuples accumulated in a
table in the worst case – in the case when all transactions during that time frame produced 1 dead tuple. Of course,
this is a very, very rough assumption, but this can give an idea about the scale and helps define "threshold" to support
the meaning of the "long-running transaction" term.

## How to prevent long-running transaction

In some cases, we might decide to prevent long-running transactions from happening at a global level, to be protected
from the negative effects described above – in this case, we decide that interrupting a long-running transaction,
leading to an error sent to one user is better than negative side effects affecting many users.

How to completely prevent long-running transactions from happening? The short answer: using just Postgres settings, you
cannot.

As of PG16 / 2023, Postgres doesn't provide a way to limit transaction duration (although there is a patch proposed,
implementing [transaction_timeout](https://commitfest.postgresql.org/45/4040/) – help test and improve it if you can).

There are two limitation settings that can help reduce chances that a long-running transaction occur, but not
eliminating the risks completely:

1) [statement_timeout](https://postgresqlco.nf/doc/en/param/statement_timeout/) – limits the maximum duration of single
   query. For web/mobile apps, set it to a low value, e.g., 30s or 15 s.

   You can find in the Postgres docs, that this is "not recommended", but that advice is not practical and I consider it
   as unproductive. We do need to limit statement_timeout globally for web and mobile apps, to be protected: the
   application code is usually limited anyway, and it's not a good situation when application reached a timeout such as
   30s, but Postgres is still processing an orphaned query. And users usually don't wait for more than a few seconds
   (Read: [What is a slow query?](https://postgres.ai/blog/20210909-what-is-a-slow-sql-query)). Those connections that
   do need a higher or even unlimited value for statement_timeout, can set it using a simple `SET` in a session (e.g.,
   connections that generate some reports, run `pg_dump`, or create indexes). Low global and overriding when needed is a
   safer approach.

2) [idle_in_transaction_session_timeout](https://postgresqlco.nf/doc/en/param/idle_in_transaction_session_timeout/) –
   sets maximum allowed idle time between queries, when in a transaction. Similar recommendations here: set it to a low
   value, 15-30s. Sessions that absolutely needed it can override the global value.

If both of these options are set to low values, it doesn't fully prevent long-running transactions from happening. For
example, if we set both of them to 30s, we might still have a transaction running for hours:

- begin;
- a query lasting <  30s
- brief delay (< 30s)
- another query lasting < 30s
- ...

– in this case, neither of the two thresholds are achieved, but we can have a transaction that hours and even days.

While there is no such a setting as `transaction_timeout` yet, we can consider alternative options to fully prevent
long-running transactions from happening:

1) A cronjob (or `pg_cron` or `pg_timetable`) record to run a "terminator" query that detects all transactions lasting
   longer than N seconds and terminates them.

   An example of such query:

   ```sql
   select clock_timestamp(), pid, query, pg_terminate_backend(pid)
   from pg_stat_activity
   where clock_timestamp() - xact_start > interval '5 minute';
   ```

   Here we need to think in advance, how to handle exclusions – e.g., `pg_dump` or sessions that build indexes. One of
   the ways here is to exclude such sessions from the scope based on their `pg_stat_activity.application_name` (setting
   `application_name` via `PGAPPNAME` is a good practice, very helpful in this case).

2) Limit it on application side. Depending on language and libraries you're using, this can be more or less difficult to
   implement. Again, we need to take care of exclusions for those sessions that do need long-running transactions.

## How to analyze long-running transactions

Getting the list of all long-running transactions is straightforward:

```sql
select clock_timestamp() - xact_start, *
from pg_stat_activity
where clock_timestamp() - xact_start > interval '1 minute'
order by clock_timestamp() - xact_start desc;
```

But for the situation described above – both `statement_timeout` and `idle_in_transaction_session_timeout` are very low,
and we still have a long-running transaction – we usually want to start sampling the states of the session that has a
long-running transaction, to understand what queries it consists of. Without such sampling, we don't have a good source
of data (queries are fast, they are usually below `log_min_duration_statement`), so we don't see them in logs.

In this case, we can apply the method described in #PostgresMarathon
[Day 11: Ad-hoc monitoring](0011_ad_hoc_monitoring.md) and sample long (> 1min) transactions every 1
second (might be worth increasing the frequency here):

```bash
while sleep 1; do
  psql -XAtc "
      copy (
        with samples as (
          select
            clock_timestamp(),
            clock_timestamp() - xact_start as xact_duration,
            * 
          from pg_stat_activity
        )
        select *
        from samples
        where xact_duration > interval '1 minute'
        order by xact_duration desc
      ) to stdout delimiter ',' csv
    " 2>&1 \
  | tee -a long_tx_$(date +%Y%m%d).log.csv
done
```
