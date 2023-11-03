Originally from: [tweet](https://twitter.com/samokhvalov/status/1707147450044297673), [LinkedIn post](https://www.linkedin.com/pulse/how-troubleshoot-speed-up-postgres-stop-restart-nikolay-samokhvalov/). 

---

# How to troubleshoot and speed up Postgres stop and restart attempts

<img src="files/0002_cover.jpg" width="600" />

#PostgresMarathon day 2. Let's discuss PostgreSQL shutdown and restarts. It's not uncommon to see quite long and even failed shutdown/restart attempts. If it's happening inside an incident troubleshooting, it often can provoke emotions and additional mistakes. 

Some very popular reasons affecting the duration of the shutdown attempt:
1. There are long-running transactions.
2. A lot of buffers are dirty (changes are applied in memory but not yet synced to disk, waiting for another checkpoint), causing long shutdown checkpoint.
3. WAL archiving (`archive_command`) is lagging.

Below, we discuss each one of these reasons and how to mitigate them.

## Reason 1: Long-running transactions

If you suspect the first reason, check if there are long-running transactions:
```sql
select
  clock_timestamp(),
  clock_timestamp() - xact_start,
  *
from pg_stat_activity
where clock_timestamp() - xact_start > interval '10 second'
order by clock_timestamp() - xact_start desc
limit 10;
```

In this case, it is usually recommended to use `SIGINT` (`pg_ctl stop -m fast`) – so-called, "fast shutdown" mode ([Postgres docs](https://postgresql.org/docs/current/server-shutdown.html)).

## Reason 2: Long shutdown checkpoint

The second reason – a lot of dirty buffers in the buffer pool – is less trivial but fortunately, is easy to mitigate. This is especially common when:
- the buffer pool size (`shared_buffers`) is large 
- checkpoint tuning was performed in favor of fewer overheads of bulk random writes and fewer full-page writes (usually meaning that `max_wal_size` and `checkpoint_timeout` are increased)
- the latest checkpoint happened quite long ago (can be seen in PG logs in `log_checkpoint = on`, which is recommended to have in most cases). 

The amount of dirty buffers is quite easy to observe, using extension pg_buffercache (standard contrib modele) and this query (may take significant time; see [the docs](https://postgresql.org/docs/current/pgbuffercache.html)):
```sql
select count(*), pg_size_pretty(count(*) * 8 * 1024)
from pg_buffercache
where isdirty;
```

If the value is large (say, several GiB), at shutdown attempt, Postgres is going to perform so-called "shutdown checkpoint", flushing dirty buffers to disk ([source code](https://gitlab.com/postgres/postgres/blob/ebf76f2753a91615d45f113f1535a8443fa8d076/src/backend/access/transam/xlog.c#L6229)). During this, it won't be processing queries, which affects downtime. Mitigation is simple – an explicit CHECKPOINT right before shutdown/restart attempt:
```sql
checkpoint;
```

This is going to help us keep shutdown checkpoint very light, decreasing downtime.

In some cases, it may make sense to issue 2 explicit CHECKPOINTs in a row, right before the shutdown attempt: if the first CHECKPOINT is heavy, it also takes time, during which new dirty buffers are accumulated due to ongoing writes – and this we mitigate with the second CHECKPOINT, keeping the shutdown checkpoint very light and fast.

To simulate situation described here:
- make sure `shared_buffers` is large (many GiB; changing it requires a restart)
- increase `max_wal_size` and `checkpoint_timeout` (change doesn't require restart): say, `'10GB'` and `'60min'` (make sure there is enough disk space in the `pg_wal` subdirectory)
- on a large table t1, perform: `set statement_timeout = '60s'; begin; delete from t1;` (it will be canceled, but lots of dirty buffers will be produced)
- install pg_buffercache and check the number of dirty buffers as mentioned above
- keep monitoring Postgres logs for checkpoint records (with `log_checkpoint=on`)

Then measure time of `pg_ctl stop -m fast` -- it is going to take significant time. Repeat the same steps with explicit CHECKPOINT right before the shutdown attempt.

This recommendation (explicit CHECKPOINT) is very important to follow in any automation involving shutdown or restart, where predictable time is needed. (For example, even if we don't aim to have minimal downtime when restarting Postgres, we still don't want it to take many minutes, potentially reaching some time limits in various places of our automation tooling.)

## Reason 3: Failing/lagging archive_command

This is an unfortunate situation: `pg_ctl stop -m fast` disconnects ongoing session but it gives the WAL archiver a chance to archive pending WALs, which affects the shutdown attempt duration (the logic is complex and can be found in these files: [postmaster.c](https://gitlab.com/postgres/postgres/blob/master/src/backend/postmaster/postmaster.c), [pgarch.c](https://gitlab.com/postgres/postgres/blob/master/src/backend/postmaster/pgarch.c)).

WAL archiving lag has to be considered a serious event because it affects the health of backups and RPO/RTO of DR. It has to be monitored and in case of significant lag (say, dozens of WALs are unarchived) it has to be considered an incident. Such monitoring should involve `pg_stat_archiver` and, ideally, information about current LSN to calculate the "lag" measured in bytes on WAL files. At least, `last_failed_wal` and `last_failed_time` should be covered by monitoring/alerting.

Interestingly, slow/failing `archive_command` can cause longer downtime during switchover/failover too – for example, this was a problem in Patroni versions older than 2.1.2, where behavior was fixed in favor of faster failover (["Release the leader lock when pg_controldata reports 'shut down'"](https://github.com/zalando/patroni/blob/master/docs/releases.rst#version-212)).

## Summary:
- use "fast mode" (`pg_ctl stop -m fast`) if you don't want existing sessions to end their work normally
- always perform an explicit CHECKPOINT before shutdown or restart attempt
- monitor `pg_stat_archiver` and ensure that `pg_stat_archiver` works without failures and lags 

---

That's it. Note that we didn't cover various timeouts (e.g., pg_ctl's option `--timeout` and wait behavior `-w`, `-W`, see [Postgres docs](https://postgresql.org/docs/current/app-pg-ctl.html)) here and just discussed what can cause delays in shutdown/restart attempts.

Hope it was helpful - as usual, [subscribe](https://twitter.com/samokhvalov/), like, share, and comment! 💙