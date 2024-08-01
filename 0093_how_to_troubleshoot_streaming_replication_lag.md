# How to troubleshoot streaming replication lag
Streaming replication in Postgres allows for continuous data replication from a primary server to standby servers, to ensure high availability and balance read-only workloads. However, replication lag can occur, leading to delays in data synchronization. This guide provides steps to troubleshoot and mitigate replication lag.

## Identifying the lag
To start investigation we need to understand where we actually have lag, on which stage of replication:
- sending WAL stream to replica via network by `walsender`
- receiving WAL stream on replica from network by `walreciever`
- writing WAL on disk on replica by `walreciever`
- applying (replaying) WAL as a recovery process

 Thus, streaming replication lag can be categorized into three types:
- **Write Lag**: The delay between when a transaction is committed on the primary and when it is written to the WAL on the standby.
- **Flush Lag**: The delay between when a transaction is written to the WAL on the standby and when it is flushed to the disk.
- **Apply (Replay) Lag**: The delay between when a transaction is flushed to the disk and when it is applied to the database on the standby.

### Analysis query
To identify the lag, use the following SQL query:
```sql
select
  pid,
  client_addr,
  application_name,
  state,
  coalesce(pg_current_wal_lsn() - sent_lsn, 0) AS sent_lag_bytes,
  coalesce(sent_lsn - write_lsn, 0) AS write_lag_bytes,
  coalesce(write_lsn - flush_lsn, 0) AS flush_lag_bytes,
  coalesce(flush_lsn - replay_lsn, 0) AS replay_lag_bytes,
  coalesce(pg_current_wal_lsn() - replay_lsn, 0) AS total_lag_bytes
from pg_stat_replication;
```

### Example
We will get something like this:
```
postgres=# select
  pid,
  client_addr,
  application_name,
  state,
  coalesce(pg_current_wal_lsn() - sent_lsn, 0) AS sent_lag_bytes,
  coalesce(sent_lsn - write_lsn, 0) AS write_lag_bytes,
  coalesce(write_lsn - flush_lsn, 0) AS flush_lag_bytes,
  coalesce(flush_lsn - replay_lsn, 0) AS replay_lag_bytes,
  coalesce(pg_current_wal_lsn() - replay_lsn, 0) AS total_lag_bytes
from pg_stat_replication;

   pid   |  client_addr   | application_name | state     | sent_lag_bytes | write_lag_bytes | flush_lag_bytes | replay_lag_bytes | total_lag_bytes
---------+----------------+------------------+-----------+----------------+-----------------+-----------------+------------------+-----------------
 3602908 | 10.122.224.101 | backupmachine1   | streaming |              0 |       728949184 |               0 |                0 |               0
 2490863 | 10.122.224.102 | backupmachine1   | streaming |              0 |       519580176 |               0 |                0 |               0
 2814582 | 10.122.224.103 | replica1         | streaming |              0 |          743384 |               0 |          1087208 |         1830592
 3596177 | 10.122.224.104 | replica2         | streaming |              0 |         2426856 |               0 |          4271952 |         6698808
  319473 | 10.122.224.105 | replica3         | streaming |              0 |          125080 |          162040 |          4186920 |         4474040
```

### How to read results
Meaning of those `_lsn` 
 - `sent_lsn`: How much WAL (lsn position) has already been sent over the network
 - `write_lsn`: How much WAL (lsn position) has been sent to the operating system (before flushing)
 - `flush_lsn`: How much WAL (lsn position) has been flushed to the disk (written on the disk)
 - `replay_lsn`: How much WAL (lsn position) has been applied (visible for queries)

So lag is a gap between `pg_current_wal_lsn` and `replay_lsn` (`total_lag_bytes`, and it's a good idea to add it to monitoring, but for troubleshooting purposes we will need all 4

- Lag on `sent_lag_bytes` means we have issues with sending the data, i.e. CPU saturated `WALsender` or overloaded network socket on the primary side
- Lag on `write_lag_bytes` means we have issues with receiving the data, i.e. CPU saturated `WALreceiver` or overloaded network socket on the replica side
- Lag on `flush_lag_bytes` means we have issues with writing the data on the disk on replica side, i.e. CPU saturated or IO contention of `WALreceiver`
- Lag `replay_lag_bytes` means we have issues with applying WAL on replica,  usually CPU saturated or IO contention of postgres process

Once we pinpointed the problem, we need to troubleshoot the process(es) on the OS level to find the bottleneck.


## Possible bottlenecks
TBD

## Additional resources
- [Streaming replication](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION) (official Postgres docs)
- [pg_stat_replication view](https://www.postgresql.org/docs/current/monitoring-stats.html#PG-STAT-REPLICATION-VIEW) (official Postgres docs)
- [Replication configuration parameters](https://www.postgresql.org/docs/current/runtime-config-replication.html) (official Postgres docs)