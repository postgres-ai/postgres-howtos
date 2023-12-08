Originally from: [tweet](https://twitter.com/samokhvalov/status/1731707060193312953), [LinkedIn post]().

---

# How to add a CHECK constraint without downtime

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Adding `CHECK` constraints can be helpful to:

- maintain better data quality
- define a `NOT NULL` constraint without downtime in PG12+
  (more [here](0060_how_to_add_a_column.md?ref_type=heads#not-null))

To add a `CHECK` constraint without downtime, we need:

1. Quickly define a constraint with the flag `NOT VALID`
2. In a separate transaction, "validate" the constraint for existing rows.

## Adding CHECK with NOT VALID

Example:

```sql
alter table t
add constraint c_id_is_positive
check (id > 0) not valid;
```

This requires a very brief `AccessExclusiveLock` lock, so on loaded systems, the command has to be executed with low
`lock_timeout` and retries (read:
[Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)).

🖋️ **Important:** Once the constraint with `NOT VALID` is in place, the new writes are checked (while old rows have not
been yet verified and some of them might violate the constraint):

```sql
nik=# insert into t select -1;
ERROR:  new row for relation "t" violates check constraint "c_id_is_positive"
DETAIL:  Failing row contains (-1).
```

## Validation

To complete the process, we need to validate the old rows:

```sql
alter table t
validate constraint c_id_is_positive;
```

This scans whole table, so for a large table, it takes long time – but this query only
acquires `ShareUpdateExclusiveLock` on the table, not blocking the sessions that run DML queries. However, a lock
acquisition attempt is going to be blocked if there is `autovacuum` running in the transaction ID wraparound prevention
mode and processing the table, or if there is another session that builds an index on this table or performs
another `ALTER` – so we need to make sure none of these heavier operations are happening before we run our `ALTER`, to
avoid excessive wait time.
