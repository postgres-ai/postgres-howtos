Originally from: [tweet](https://twitter.com/samokhvalov/status/1707466169245171773), [LinkedIn post](https://www.linkedin.com/pulse/how-troubleshoot-long-postgres-startup-nikolay-samokhvalov/). 

---

# How to troubleshoot long Postgres startup

<img src="files/0003_cover.png" width="600" />

#PostgresMarathon day 3. In [the previous post](./0002_how_to_troubleshoot_and_speedup_postgres_restarts.md), we discussed how to quickly stop or restart PostgreSQL. Now it's time to discuss what to do if you are trying to start your server but see this:
```
FATAL:  the database system is not yet accepting connections
DETAIL:  Consistent recovery state has not been yet reached.
```

What NOT to do (common patterns for non-experts):
- start worrying or waiting for a long time without understanding how long it might take
- attempt to stop/start again, and again

What to do:
1. Keep calm
2. Understand your settings and workload
3. Understand and observe the REDO progress

Below each step is discussed in detail.

## 1. Keep calm
The most difficult part (if it's production incident, there is time pressure, etc.), but it is necessary. To help with it, let's understand what's happening.

The message above means that Postgres is starting but has not yet finished REDO – replaying WAL records since the latest successful checkpoint.

Log example showing that indeed, REDO started:
```
2023-09-28 10:17:06.864 PDT [83002] LOG:  database system was interrupted; last known up at 2023-09-27 14:49:18 PDT
2023-09-28 10:17:07.126 PDT [83002] LOG:  database system was not properly shut down; automatic recovery in progress
2023-09-28 10:17:07.130 PDT [83002] LOG:  redo starts at 26/6504A218
```

One of the biggest causes of frustration can be a lack of understanding of what's happening and an inability to answer a simple question: "Is it doing anything?" Below, we'll address these aspects.

## 2. Understand your settings and workload

### 2a. Check max_wal_size and checkpoint_timeout
The settings that matter the most here are related to checkpoint tuning. If `max_wal_size` and `checkpoint_timeout` are tuned so checkpoints happen less often, Postgres needs more time to reach a consistency point, if shutdown was not clean, without successful shutdown checkpoint (e.g., VM restarted) or if you're restoring from backups. To learn more about this:
- [official docs](https://postgresql.org/docs/current/sql-checkpoint.html) (official docs)
- [WAL and checkpoint tuning](https://postgres.fm/episodes/wal-and-checkpoint-tuning) (Postgres.fm podcast) 

In other words, if you observe longer startup time, it's probably because the server was tuned to write less to WAL and sync buffers less often during heavy loads at normal times – that tuning comes for the price of longer startup time, and this is exactly what you're dealing with.

### 2b. Understand the actual checkpoint behavior
It is definitely recommended to have `log_checkpoint = on`. Its default is `'off'` in Postgres 14 and older, and `'on'` in PG15+.

With `log_checkpoint`, you can see checkpoint information in the logs, and understand how frequent they are and how much data is being written. An example:
```
2023-09-28 10:17:17.798 PDT [83000] LOG:  checkpoint starting: end-of-recovery immediate wait
2023-09-28 10:17:46.479 PDT [83000] LOG:  checkpoint complete: wrote 465883 buffers (88.9%); 0 WAL file(s) added, 0 removed, 0 recycled; write=28.667 s, sync=0.001 s, total=28.681 s; sync files=6, longest=0.001 s, average=0.001 s; distance=5241900 kB, estimate=5241900 kB
```

In Postgres 16+, you can also see checkpoint and REDO LSNs in these logs, which is going to be very helpful for REDO progress monitoring (see below).

If you don't know what LSN is, read the following:
- [pg_lsn Type](https://postgresql.org/docs/current/datatype-pg-lsn.html) (official doc)
- [Postgres WAL Files and Sequence Numbers](https://crunchydata.com/blog/postgres-wal-files-and-sequuence-numbers)
- [WAL, LSN and File Names](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames)

### 2c. Understand workload
WALs are stored in `$PGDATA/pg_wal`, typically 16 MiB in size (this can be adjusted – for example, RDS has changed it to 64 MiB). Heavily loaded systems can have many WALs created per second, and producing many TiB of WAL data per day.

- Inspecting pg_wal subdirectory in PGDATA helps to understand how many WALs are created per minute
- If current LSN and its growth rates are present in your monitoring, it is also helpful – if you take two LSNs for two points in time, you can subtract one from another, and you'll get the distance in bytes, for example:
    ```
    nik=# select pg_size_pretty('39/EF000000'::pg_lsn - '32/AA000000'::pg_lsn);
     pg_size_pretty
    ----------------
     29 GB
    (1 row)
    ```
- Similarly, you can get LSNs corresponding to backups created by your backup tool, and calculate the difference between them

## 3. Understand and observe the REDO progress
To understand the progress, we need to be able to get several values:
- current position of the REDO process (LSN currently being replayed)
- target LSN – the consistency point, where REDO is going to finish
- (optionally) starting LSN – where/when we have started

Unfortunately, obtaining these values is not so easy – Postgres doesn't report them in logs (to new/potential hackers: it is a good idea to implement this, by the way). And we cannot use SQL to get anything because `FATAL:  the database system is not yet accepting connections`.

If you manage Postgres yourself, you can do this to determine the values, understand and monitor the progress.

First, see where we are:
```
❯ ps ax | grep 'startup recovering'
98786   ??  Us     0:15.81 postgres: startup recovering 000000010000004100000018
```

A little bit later:
```
❯ ps ax | grep 'startup recovering' | grep -v grep
99887   ??  Us     0:02.29 postgres: startup recovering 000000010000004500000058
```

– as we can see, the position is changing, so REDO is progressing. This is already useful and should give some relief. 

Now, we cannot use SQL, but we can use `pg_controldata` to see meta information about the cluster's state (you need to specify PGDATA location, using `-D`):

```
❯ /opt/homebrew/opt/postgresql@15/bin/pg_controldata -D /opt/homebrew/var/postgresql@15 | grep Latest | grep -e location -e WAL
Latest checkpoint location:           48/E10B8B50
Latest checkpoint's REDO location:    45/1772EA98
Latest checkpoint's REDO WAL file:    000000010000004500000017
```

In Postgres 16, additionally, you can see where (at which LSN) and when (timestamp) the REDO process has been started, if log_checkpoint=on (recommended). Example:
```
2023-09-28 01:23:32.613 UTC [81] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.003 s, sync=0.002 s, total=0.025 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=97 kB; lsn=0/1274EF8, redo lsn=0/1274EC1
```

If we are dealing with a replica and/or restoring it from backups, we can use this value to understand when REDO is going to finish:
```
❯ /opt/homebrew/opt/postgresql@15/bin/pg_controldata -D /opt/homebrew/var/postgresql@15 | grep 'Minimum recovery ending location'
Minimum recovery ending location:     4A/E0FFE518
```

Doing the math (good explanation can be found in this article: https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames):

- Current position `000000010000004500000058` -- LSN `45/58000000`
- Our target position `4A/E0FFE518`
- Starting point `45/1772EA98`

Now in psql (on any working Postgres):
```
nik=# select pg_size_pretty(pg_lsn '4A/E0FFE518' - pg_lsn '45/58000000');
 pg_size_pretty
----------------
 22 GB
(1 row)

nik=# select pg_size_pretty(pg_lsn '45/58000000' - '45/1772EA98');
 pg_size_pretty
----------------
 1033 MB
(1 row)
```

– the first value is what's left, the second value is what's already done. Here, we see that we've already replayed ~1 GiB, and ~22 GiB are left. Analyzing timestamps in Postgres logs and current time and assuming that REDO is performed at constant speed (this assumption is rough but it's ok for rough estimate, especially if we re-estimating multiple times while observing the process), we can have an estimate of how much left to wait till consistency point and ability for Postgres to accept connections.

If we're dealing with a crashed Postgres, then normally `pg_controldata` doesn't provide `Minimum recovery ending location` (showing `0/0`). In this case, we can check `$PGDATA/pg_wal` to understand how much left to replay, ordering files by creation time. This works under assumption that when crashed, Postgres has WALs that need to be replayed in pg_wal, and the "tail" of those WALs is all we need. For example:

```
❯ ls -la /opt/homebrew/var/postgresql@15/pg_wal | grep 0000 | tail  -3
-rw-------     1 nik  admin  16777216 Sep 28 10:55 000000010000004A000000DF
-rw-------     1 nik  admin  16777216 Sep 28 10:55 000000010000004A000000E0
-rw-------     1 nik  admin  16777216 Sep 28 11:03 000000010000004A000000E1
```

– the latest file is `000000010000004A000000E1`, hence we can take `4A/E100000` as a rough estimate where we'll finish with the REDO process.

---
Bonus: how to simulate long startup / REDO time:
1. Increase the distance between checkpoints raising `max_wal_size` and `checkpoint_timeout` (say, `'100GB'` and `'60min'`)
2. Create a large table `t1` (say, 10-100M rows): `create table t1 as select i, random() from generate_series(1, 100000000) i;`
3. Execute a long transaction to data from `t1` (not necessary to finish it): `begin; delete from t1;`
4. Observe the amount of dirity buffers with extension `pg_buffercache`:
   -  create extension `pg_buffercache`;
   - `select isdirty, count(*), pg_size_pretty(count(*) * 8 * 1024) from pg_buffercache  group by 1 \watch`
5. When the total size of dirty buffers reaches a few GiB, intentionally crash your server, sending `kill -9 <pid>` using PID of any Postgres backend process.
6. Ensure Postgres is down: `ps ax | grep postgres`
7. Start Postgres (e.g., `pg_ctl -D <PGDATA> start`)
8. Check Postgres logs and ensure that REDO is happening.
   
---

That's it! I hope this helps keep you calm and allows you to wait consciously for your Postgres to open the gates to fun SQL queries. Hopefully, fewer folks will be stressed out during long startup times.

Note that this post is a How-To draft; I wrote it quickly, and it might contain some inaccuracies. I'm going to upload all my How-To drafts to a Git repo soon to keep them open, public, and welcoming of fixes & improvements. Stay tuned!

As usual, if you find this helpful, please [subscribe](https://twitter.com/samokhvalov), like, share, and comment! 💛
