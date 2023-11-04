Originally from: [tweet](https://twitter.com/samokhvalov/status/1717773398586298703), [LinkedIn post]().

---

# How to troubleshoot a growing pg_wal directory

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Directory `$PGDATA/pg_wal` contains WAL files. WAL (Write-Ahead Log) is a core mechanism used for backups, recovery, and
both physical and logical replication.

In certain cases, `pg_wal` size keeps growing, and this can become concerning due to increasing risks to be out of disk
space.

Here are the things to check to troubleshoot a growing `pg_wal` directory.

## Step 1: check replication slots

Unused or lagging replication slots keep WALs from being recycled, hence the `pg_wal` directory size grows.

Check on the primary:

```sql
  select
    pg_size_pretty(pg_current_wal_lsn() - restart_lsn) as lag,
    slot_name,
    wal_status,
    active
  from pg_replication_slots
  order by 1 desc;
```

Reference doc: [The view pg_replication_slots](https://postgresql.org/docs/current/view-pg-replication-slots.html)

- If there are inactive replication slots, consider dropping them to prevent reaching 100% of used disk space. Once the
  problematic slot(s) are dropped, Postgres will remove old WALs.
- Alternatively, consider using 
  [max_slot_wal_keep_size (PG13+)](https://postgresqlco.nf/doc/en/param/max_slot_wal_keep_size/).

## Step 2: check if `archive_command` works well

If `archive_mode` and `archive_command` are configured to archive WALs (e.g., for backup purposes), but 
`archive_command` is failing (returns non-zero exit code) or lagging (WAL generation rates are higher than the speed of
archiving), then this can be another reason of `pg_wal` growth.

How to monitor and troubleshoot it:

- check [pg_stat_archiver](https://postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ARCHIVER-VIEW)
- check Postgres logs (e.g., check for `archive command failed with exit code 1`)

Once the problem is identified, the `archive_command` needs to be either fixed or sped up (e.g., `wal_compression = on`,
`max_wal_size` increased to have less WAL data generated; and, at the same time, use lighter compression in the archiver
tool -- this depends on the tool used in `archive_command`; e.g., WAL-G support many options for compression, more or
less CPU intensive).

The next two steps are to be considered as additional, since their effects on the `pg_wal` size growth are limited – 
they can cause only certain amount of extra WALs being kept in `pg_wal` 
(unlike the first two reasons we just discussed).

## Step 3: check `wal_keep_size`

In some cases, `wal_keep_size` ([PG13+ docs](https://postgresqlco.nf/doc/en/param/wal_keep_size/); in PG12 and older, 
see `wal_keep_segments`) is set to a high value. When slots are used, it's not generally needed – this is an older (than
slots) mechanism to avoid situations when some WAL is deleted and a lagging replica cannot catch up.

## Step 4: check `max_wal_size` and `checkpoint_timeout`

When a successful checkpoint happens, Postgres can delete old WALs. In some cases, if checkpoint tuning was performed in
favor of less frequent checkpoints, this can cause more WALs to be stored in `pg_wal` than one could expect. In this 
case, if it's a problem for disk space (specifically important on smaller servers), reconsider `max_wal_size` and 
`checkpoint_timeout` to lower values. In some cases, it also can make sense to run an explicit manual `CHECKPOINT`, to
allow Postgres clean up some old files right away.

## Summary

Most important:

1. Check for unused or lagging replication slots
2. Check for failing or lagging `archive_command`

Additionally:

3. Check `wal_keep_size`
4. Check `max_wal_size` and `checkpoint_timeout`
