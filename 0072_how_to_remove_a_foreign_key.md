Originally from: [tweet](https://twitter.com/samokhvalov/status/1732855546141933694), [LinkedIn post]().

---

# How to remove a foreign key

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Removing a foreign key (FK) is straightforward:

```
alter table messages
drop constraint fk_messages_users;
```

However, under heavy load, this is not a safe operation, due to the reasons discussed in
[zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries):

- this operation needs a brief `AccessExclusiveLock` on *both* tables
- if at least one of the locks cannot be quickly acquired, it waits, potentially blocking other sessions (including
  `SELECT`s), to both tables.

To solve this problem, we need to use a low lock_timeout and retries:

```sql
set lock_timeout to '100ms';
alter ... -- be ready to fail and try again.
```

The same technique needs to be used in the first step of the operation of new FK creation, when we define a new FK with
the `NOT VALID` option, as was discussed in [day 70, How to add a foreign key](0070_how_to_add_a_foreign_key.md).

See also: [day 71, How to understand what's blocking DDL](0071_how_to_understand_what_is_blocking_ddl.md).
