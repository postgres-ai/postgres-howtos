Originally from: [tweet](https://twitter.com/samokhvalov/status/1727344499943493900), [LinkedIn post]().

---

# How to convert a physical replica to logical

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

In some cases, it might be beneficial to convert an existing regular asynchronous physical replica to logical, or to
create a new physical replica first, and then convert it to logical.

This approach:

- on the one hand, eliminates the need to execute initial data load step that can be fragile and quite stressful in case
  of large, heavily-loaded DB, but
- on another, the logical replica created in such way has everything that the source Postgres instance has.

So, this method suits better in case when you need all the data from the source be presented in the logical replica
you're creating, and it is extremely useful if you work with very large, heavily-loaded clusters.

The steps below are quite straightforward. In this case, we use a physical replica that replicates data immediately from
the primary via streaming replication `primary_conninfo` and replication slot (e.g. under Patroni's control), not
involving cascaded replication (although it's possible to implement too).

## Step 1: have a physical replica for conversion

Choose a physical replica to convert, or create a new one using `pg_basebackup`, recovering from backups, or creating it
from a cloud snapshot.

Make sure this replica is not used by regular users while we're converting it.

## Step 2: ensure the requirements are met

First, ensure that the settings are prepared for logical replication, as described
in the [logical replication config](https://postgresql.org/docs/current/logical-replication-config.html).

Primary settings:

- `wal_level = 'logical'`
- `max_replication_slots > 0`
- `max_wal_senders > max_replication_slots`

On the physical replica we are going to convert:

- `max_replication_slots > 0`
- `max_logical_replication_workers > 0`
- `max_worker_processes >= max_logical_replication_workers + 1`

Additionally:

- the replication lag is low;
- every table has a PK or have
  [REPLICA IDENTITY FULL](https://postgresql.org/docs/current/sql-altertable.html#SQL-ALTERTABLE-REPLICA-IDENTITY);
- `restore_command` is not set on the replica we'll use (if it is, temporarily set its value to an empty string);
- temporarily, increase `wal_keep_size` (PG13+; in PG12 or older, `wal_keep_segments`) on the primary to a value
  corresponding to a few hours of WAL generation.

## Step 3: stop physical replica

Shut down physical replica and keep it down during the next step. This is needed so its position is guaranteed to be in
the past compared to the logical slot we're going to create on the primary.

## Step 4: create publication, logical slot, and remember its LSN

On the primary:

- issue a manual `CHECKPOINT`;
- create publication;
- create a logical slot and *remember its LSN position*;

Example:

```sql
checkpoint;

create publication my_pub for all tables;

select lsn
from pg_create_logical_replication_slot(
  'my_slot',
  'pgoutput'
);
```

It is important to remember the `lsn` value from the last command – we'll be using it further.

## Step 5: let the physical replica catch up

Reconfigure the physical replica:

- `recovery_target_lsn` – set it to the LSN value we've got from the previous step
- `recovery_target_action = 'promote'`
- `restore_command`, `recovery_target_timeline`, `recovery_target_xid`, `recovery_target_time`, `recovery_target_name`
  are not set or empty

Now, start the physical replica. Monitor its lag and how the replica catches up reaching the LSN we need and then
auto-promotes. This can take some time. Once it's done, check it:

```sql
select pg_is_in_recovery();
```

- must return `f`, meaning that this node is now a primary itself (a clone) with position, corresponding to the position
  of the replication slot on the source node.

## Step 6: create subscription and start logical replication

Now, of the freshly created "clone", create logical subscription with `copy_data = false` and `create_slot = false`:

```sql
create subscription 'my_sub'
connection 'host=.. port=.. user=.. dbname=..'
publication my_pub
with (
  copy_data = false,
  create_slot=false,
  slot_name = 'my_slot'
);
```

Ensure that replication is now active – check it on the source primary:

```sql
select * from pg_replication_slots;
```

– the field `active` must be `t` for our slot.

## Finalize

- Wait until the logical replication lags fully caught up (occasional acute spikes are OK).
- Return `wal_keep_size` (`wal_keep_segments`) to its original value on the primary.

## Additional notes

Here we used a single publication and logical slot in this recipe. It is possible to use multiple slots, slightly
adjusting the procedure. But if you choose to do so, keep in mind the potential complexities of the use of multiple
slots/publications, first of all, these:

- not guaranteed referential integrity on the logical replica (occasional temporary FK violation),
- more fragile procedure of publication creation (creation of a publication `FOR ALL TABLES` doesn't require table-level
  locks; but when we use multiple publications and create publication for certain tables, table-level locks are
  required – however, this is just `ShareUpdateExclusiveLock`,
  per [this comment on PostgreSQL source code](https://github.com/postgres/postgres/blob/1b6da28e0668eb977dcab6987d192ddedf32b752/src/backend/commands/publicationcmds.c#L1550)).

And in any case:

- make sure you are prepared to deal with the restrictions of logical replication for your version (e.g.,
  [for PG16](https://postgresql.org/docs/16/logical-replication-restrictions.html));
- if you consider using this approach to perform a major upgrade, avoid running `pg_upgrade` on the already-converted
  node – it may be not safe
  (see: [pg_upgrade and logical replication](https://postgresql.org/message-id/flat/20230217075433.u5mjly4d5cr4hcfe%40jrouhaud)).
