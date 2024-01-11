Originally from: [tweet](https://twitter.com/samokhvalov/status/1732056107483636188), [LinkedIn post]().

---

# How to add a foreign key

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Adding a foreign key (FK) is straightforward:

```sql
alter table messages
add constraint fk_messages_users
foreign key (user_id)
references users(id);
```

However, this operation requires locks on both tables involved:

1. `ShareRowExclusiveLock`, `RowShareLock`, and `AccessShareLock` on the referenced table, in this example it's
   `users`  (plus `AccessShareLock` on its primary key, PK). This blocks any data modifications to `users` (`UPDATE`,
   `DELETE`, `INSERT`), as well as DDL.

2. `ShareRowExclusiveLock` and `AccessShareLock` to the referencing table, in this example `messages`
   (plus, `AccessShareLock` to its PK). Again, this blocks writes to this table, and DDL.

And to ensure that the existing data doesn't violate the constraint, full table scans are needed – so the more data the
tables have, the longer this implicit scan is going to take. During which, the locks are going to block all writes and
DDL to the table.

To avoid downtime, we need to create the FK in three steps:

1. Quickly define a constraint with the flag `NOT VALID`.
2. For the existing data, if needed, fix rows that would break the FK.
3. In a separate transaction, `validate` the constraint for existing rows.

## Step 1: Add FK with NOT VALID

Example:

```sql
alter table messages
add constraint fk_messages_users
foreign key (user_id)
references users(id)
not valid;
```

This requires a very brief `ShareRowExclusiveLock` and `AccessShareLock` on both tables, so on loaded systems, it is
still recommended to execute this with low `lock_timeout` and retries (read:
[Zero-downtime database schema migrations](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)),
to avoid lock queue blocking writes to the tables.

🖋️ **Important:** once the constraint with `NOT VALID` is in place, new writes are checked (while old rows have not
been yet verified and some of them might violate the constraint):

```sql
nik=# \d messages
              Table "public.messages"
 Column  |  Type  | Collation | Nullable | Default
---------+--------+-----------+----------+---------
 id      | bigint |           | not null |
 user_id | bigint |           |          |
Indexes:
    "messages_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "fk_messages_users" FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID

nik=# insert into messages(id, user_id) select 1, -1;
ERROR:  insert or update on table `messages` violates foreign key constraint "fk_messages_users"
DETAIL:  Key (user_id)=(-1) is not present in table `users`.
```

## Step 2: Fix existing data if needed

Now, with the FK created with `NOT VALID`, we know that Postgres already checks all the new data against the new
constraint, but for the old data, some rows might still be violating it. Before the next step, it makes sense to ensure
there are no old rows violating our new FK. It can be done using this query:

```sql
select id
from messages
where
  user_id not in (
    select id from users
  );
```

This query scans the whole `messages` table, so it will take significant time. It is worth ensuring that `users` is
accessed via PK here (depends on the data volumes and planner settings).

The rows identified by this query will block the next step, so they need to either be deleted or changed to avoid the FK
violation.

## Step 3: Validation

To complete the process, we need to `validate` the old rows in a separate transaction:

```sql
alter table messages
validate constraint fk_messages_users;
```

If the tables are large, this `ALTER` is going to take significant time. However, it only acquires
`ShareUpdateExclusiveLock` and `AccessShareLock` on the referencing table (`messages` in this example).

Therefore, it doesn't block `UPDATE` / `DELETE` / `INSERT`, but it conflicts with DDL and `VACUUM` runs. On the
referenced table (`users` here), `AccessShareLock` and `RowShareLock` are acquired.

As usual, if `autovacuum` processes this table in the transaction ID wraparound prevention mode, it won't yield – so
before running this, make sure there is no `autovacuum` running in this mode or DDL in progress.
