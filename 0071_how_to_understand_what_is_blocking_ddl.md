Originally from: [tweet](https://twitter.com/samokhvalov/status/1732486201830240283), [LinkedIn post]().

---

# How to understand what's blocking DDL

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

In [day 60](0060_how_to_add_a_column.md), we discussed how to add a column and that under load you need
low `lock_timeout` and retries (also
see: [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)).

There we used `max_attempts` set to `1000` and this is probably too many, but interesting question is: how to understand
what's blocking our DDL, if it doesn't succeed after many retries?

The first thing to do is to enable `log_lock_waits`. In this case, after `deadlock_timeout` (`1s` by default; and this
setting defines when deadlock detection happens), you'll see some information about our blocked session.

But we don't want to start with `lock_timeout` more than `1s` – it would be too invasive. Solution is to set a lower
`deadlock_timeout` right in session. Example (assuming that there is another session just read from table `t` and keeps
transaction open):

```sql
postgres=# set lock_timeout to '100ms';
SET

postgres=# set deadlock_timeout to '50ms';
SET

postgres=# set log_lock_waits to 'on';
SET

postgres=# alter table t1 add column new_c int8;
ERROR:  canceling statement due to lock timeout
```

In the logs we have two entries – one after ~`50ms` (`deadlock_timeout`) with some useful details about the waiting on
lock acquisition, and another when our statement fails after `100ms`:

```
2023-12-06 19:12:21.823 UTC [197] LOG:  process 197 still waiting for AccessExclusiveLock on relation 16384 of database 13481 after 54.022 ms
2023-12-06 19:12:21.823 UTC [197] DETAIL:  Process holding the lock: 211. Wait queue: 197.
2023-12-06 19:12:21.823 UTC [197] STATEMENT:  alter table t1 add column new_c int8;
2023-12-06 19:12:21.874 UTC [197] ERROR:  canceling statement due to lock timeout
2023-12-06 19:12:21.874 UTC [197] STATEMENT:  alter table t1 add column new_c int8;
```

Unfortunately, the log message that was written when we achieved `deadlock_timeout` (`50ms` in this case) doesn't report
details about the blocker – just `PID` (`Process holding the lock: 211`). If we're lucky, we might have details about
this session around the timestamp we look at in the Postgres logs – we should just look around the timestamp when
failure occurred. We might find that it's `autovacuum` running in the transaction ID wraparound prevention mode, some
DDL (if we log DDL with `log_statement='ddl'`), or some long-running query that has reached `log_min_duration_statement`
(worth setting it to `1s` or a lower value controlling how much we actually log to avoid the "observer effect"), and got
logged. But in many cases, there is nothing else in the logs, and we need another solution.

In this case, we can do this:

1) Use a custom `application_name` value for sessions that run DDL, to more easily distinguish them
   in `pg_stat_activity` – simply setting it in `PGAPPNAME` or just via `SET`:

    ```sql
    set application_name = 'ddl_runner';
    ```

2) In a separate session, run an observer query. For example, this anonymous `DO` block with PL/pgSQL code:

    ```sql
    do $$
    declare
      i int;
      rec record;
      wait_more boolean := false;
    begin
      for i in select generate_series(1, 2000) loop
        for rec in
          select
            pid,
            left(query, 50) as query_left50,
            *
          from pg_stat_activity
          where
            application_name = 'ddl_runner'
            and wait_event_type = 'Lock'
        loop
          raise log 'DDL session blocked. Session info: pid=%, query_left50=%, wait=%:%. Blockers: %',
            rec.pid,
            rec.query_left50,
            rec.wait_event_type,
            rec.wait_event,
            (
              select
                array_agg(json_build_object(
                  'pid', pid,
                  'state', state,
                  'wait', (wait_event_type || ':' || wait_event),
                  'query_l50', left(query, 50)
                ))
              from pg_stat_activity
              where array[pid] <@ pg_blocking_pids(rec.pid)
            );

            wait_more := true;
        end loop;

        if wait_more then
          perform pg_sleep(1);

          wait_more := false;
        else
          perform pg_sleep(0.05);
        end if;
      end loop;
    end $$;
    ```

   This observer code will report something like this, if DDL session waits more than 50ms:

   > 2023-12-06T19:37:35.746363053Z 2023-12-06 19:37:35.746 UTC [237] LOG:  DDL session blocked. Session info: pid=197, query_left50=alter table t1 add column new_c int8;, wait=Lock
   >
   > :relation. Blockers: {"{\"pid\" : 211, \"state\" : \"idle in transaction\", \"wait\" : \"Client:ClientRead\", \"query_l50\" : \"select from t1 limit 0;\"}"}

   This should be enough for troubleshooting of failing DDL attempts.

A couple of more notes:

- Instead of anonymous DO block, it's probably better to wrap PL/pgSQL code into a function, so the `CONTEXT` and
  `STATEMENT` parts of the log messages don't consume too much space.
- Such a function then can be invoked by `pg_cron`, so you have a permanent observability tool. But it should be taken
  into account that it's a whole backend that runs this, so it might be not wise to have it constantly running –
  instead, it could wake up from time to time only during periods when we know some DDL can be executed.
