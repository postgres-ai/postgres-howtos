Originally from: [tweet](https://twitter.com/samokhvalov/status/1712714779612336128), [LinkedIn post](...).

---

# How to determine the replication lag

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## On primary / standby leader

When connected to a primary (or standby leader in case of cascaded replication), one can use `pg_stat_replication`:

```
nik=# \d pg_stat_replication
                    View "pg_catalog.pg_stat_replication"
      Column      |           Type           | Collation | Nullable | Default
------------------+--------------------------+-----------+----------+---------
 pid              | integer                  |           |          |
 usesysid         | oid                      |           |          |
 usename          | name                     |           |          |
 application_name | text                     |           |          |
 client_addr      | inet                     |           |          |
 client_hostname  | text                     |           |          |
 client_port      | integer                  |           |          |
 backend_start    | timestamp with time zone |           |          |
 backend_xmin     | xid                      |           |          |
 state            | text                     |           |          |
 sent_lsn         | pg_lsn                   |           |          |
 write_lsn        | pg_lsn                   |           |          |
 flush_lsn        | pg_lsn                   |           |          |
 replay_lsn       | pg_lsn                   |           |          |
 write_lag        | interval                 |           |          |
 flush_lag        | interval                 |           |          |
 replay_lag       | interval                 |           |          |
 sync_priority    | integer                  |           |          |
 sync_state       | text                     |           |          |
 reply_time       | timestamp with time zone |           |          |
```

This view contains information for both physical (only those using streaming, not WAL shipping) and logical replicas.
The lag values here are measured in bytes (columns `***_lsn`) or in time intervals (columns `***_lag`), and multiple
steps can be observed for each replication stream.

To analyze the `LSN` values in this system view, we need to subtract them from current `LSN` values on the server we're
connected to:

1) if it's primary (`pg_is_in_recovery()` returns `false`), then use `pg_current_wal_lsn()`
2) otherwise (standby leader), use `pg_last_wal_replay_lsn()`

To subtract, we can use the function `pg_wal_lsn_diff()` or the `-` operator.

Docs: [Monitoring pg_stat_replication view](https://postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW).

Examples of queries to view lags:

- [Netdata](https://github.com/netdata/netdata/blob/9edcad92d83dba359f5eb6c06d0741b50030edcf/collectors/python.d.plugin/postgres/postgres.chart.py#L486)

- [Postgres exporter for Prometheus](https://github.com/prometheus-community/postgres_exporter/blob/2a5692c0283fddf96e776cc73c2fc0d5caed1af6/cmd/postgres_exporter/queries.go#L46)

## On a physical replica

When connected to a physical replica (standby), to get its lag as a time interval:

```sql
select now() - pg_last_xact_replay_timestamp();
```

In some cases, `pg_last_xact_replay_timestamp()` may return `NULL`:

- if the standby server has just started and hasn't replayed any transactions,
- if there are no recent transactions on the primary.

This behavior of `pg_last_xact_replay_timestamp()` might lead to wrong conclusions that standby is lagging and that
replication is not healthy – this isn't uncommon in low-activity setups (e.g., non-production environments).

Docs: [pg_last_xact_replay_timestamp](https://postgresql.org/docs/current/functions-admin.html#id-1.5.8.33.6.3.2.2.4.1.1.1).

## Logical replication

The lag can be observed from the primary (in the contest of logical replication, it's usually called "publisher).

In addition to `pg_stat_replication` already discussed above, `pg_replication_slots` can also be used:

```sql
select
    slot_name,
    pg_current_wal_lsn() - confirmed_flush_lsn as lag_bytes
from pg_replication_slots;
```

## Hybrid case: logical & physical

In some cases, you might need to deal with a combination of logical and physical replication. For example, consider the
case:

- Cluster A: a regular cluster of 3 nodes (primary + 2 physical standbys).
- Cluster B: also primary + 2 physical nodes, and the primary connected to the cluster A's primary via logical
  replication

This is a typical situation when a complex change (e.g., a major upgrade) is performed involving logical replication. At
some point, you might want to redirect a portion of the read-only traffic from cluster A's standbys to cluster B's
standbys.

In this case, if we need to understand how each of the nodes is lagging, and we want the code to work well with all
nodes, it can be tricky.

Here is how it could be solved
(credits: [Dylan Griffith from GitLab](https://gitlab.com/gitlab-org/gitlab/-/merge_requests/121621)), assuming we can
get information from both the
node we're analyzing and the cluster A's primary (keeping all the comments above in mind):

1. First, get `LSN` on the primary:
   ```sql
   select pg_current_wal_lsn() as primary_lsn;
   ```

2. Then obtain the `LSN` location of the observed node, and use it to calculate the lag value in bytes:
   ```sql
   with current_node as (
     select case
       when exists (select from pg_replication_origin_status) then (
         select remote_lsn
         from pg_replication_origin_status
       )
       when pg_is_in_recovery() then pg_last_wal_replay_lsn()
       else pg_current_wal_lsn()
     end as lsn
   )
   select lsn – {{primary_lsn}} as lag_bytes
   from current_node;
   ```
