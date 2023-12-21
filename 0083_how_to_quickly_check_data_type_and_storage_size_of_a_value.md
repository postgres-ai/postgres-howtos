Originally from: [tweet](https://twitter.com/samokhvalov/status/1737019068987937049), [LinkedIn post]().

---

# How to change a Postgres parameter

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Follow these steps if you need to change a Postgres parameter (a.k.a., GUC, Grand Unified Configuration) for permanent
effect.

Docs: [Setting Parameters](https://postgresql.org/docs/current/config-setting.html.)

## 1) Understand if a restart is needed

Two ways to quickly check if a restart is needed:

- Use [postgresqlco.nf](https://postgresqlco.nf) and look at the `Restart: xxx` field. For example, for
  [max_wal_size](https://postgresqlco.nf/doc/en/param/max_wal_size/), `Restart: false`, while for
  [shared_buffers](https://postgresqlco.nf/doc/en/param/shared_buffers), it's `true`.

- Check `context` in `pg_settings` – if it's `postmaster`, then a restart is needed, otherwise it's not (small
  exception: values `internal` – such parameters cannot be changed at all). For example:

  ```sql
  nik=# select name, context
  from pg_settings
  where name in ('max_wal_size', 'shared_buffers');
        name      |  context
  ----------------+------------
   max_wal_size   | sighup
   shared_buffers | postmaster
  (2 rows)
  ```

## 2) Perform the change

Apply the change in Postgres config files (`postgresql.conf` or its dependencies, if `include` directive is used). It's
advisable to `ALTER SYSTEM` unless necessary, because it might lead to confusion in the future (it writes
to `postgresql.auto.conf` and later it can be easily overlooked; see also
[this discussion](https://postgresql.org/message-id/flat/CA%2BVUV5rEKt2%2BCdC_KUaPoihMu%2Bi5ChT4WVNTr4CD5-xXZUfuQw%40mail.gmail.com))

## 3) Apply the change

If a restart is required, restart Postgres. If you forget to do it, you can later detect the situation of un-applied
changes by looking at `pending_restart` in `pg_settings`.

If a restart is not needed, execute under a superuser:

```sql
select pg_reload_conf();
```

Alternatively, you can use one of these methods:

- `pg_ctl reload $PGDATA`
- send a `SIGHUP` to the postmaster process, e.g.:

  ```bash
  kill -HUP $(cat "${PGDATA}/postmaster.pid")
  ```

When a non-restart-required change is applied, you'll see something like this in the Postgres log:

```
LOG: received SIGHUP, reloading configuration files
LOG: parameter "max_wal_size" changed to "2GB"
```

## 4) Verify the change

Use `SHOW` or `current_setting(...)` to ensure that the change is applied, e.g.:

```sql
nik=# show max_wal_size;
 max_wal_size
--------------
 2GB
(1 row)
```

or

```sql
nik=# select current_setting('max_wal_size');
 current_setting
-----------------
 2GB
(1 row)
```

## Bonus: database-, user-, and table-level settings

Settings with `context` in `pg_settings` being `user` or `superuser` can be adjusted at database or user level, e.g.:

```sql
alter database verbosedb set log_statement = 'all';
alter user hero set statement_timeout = 0;
```

The result of this can be reviewed by looking at `pg_db_role_setting`:

```sql
nik=# select * from pg_db_role_setting;
 setdatabase | setrole |       setconfig
-------------+---------+-----------------------
       24580 |       0 | {log_statement=all}
           0 |   24581 | {statement_timeout=0}
(2 rows)
```

Some settings can also be adjusted at individual table level, when using `CREATE TABLE` or `ALTER TABLE`, see
[CREATE TABLE / Storage Parameters](https://postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS).
Keep in mind naming deviation: `autovacuum_enabled` enables or disables the autovacuum daemon for a particular table,
while the global setting name is simply `autovacuum`.
