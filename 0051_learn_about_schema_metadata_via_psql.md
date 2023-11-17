Originally from: [tweet](https://twitter.com/samokhvalov/status/1725064379933319287), [LinkedIn post]().

---

# Learn how to work with schema metadata by spying after psql

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## Where to start when unsure

[psql Docs](https://postgresql.org/docs/current/app-psql.html)

`psql` has internal help – command `\?`; it's worth remembering and using it as a starting point when working
with `psql`.

When unsure about a function name, it can be helpful to use search `\df *keyword*`. For example:

```
nik=# \df *clock*
                                  List of functions
   Schema   |      Name       |     Result data type     | Argument data types | Type
------------+-----------------+--------------------------+---------------------+------
 pg_catalog | clock_timestamp | timestamp with time zone |                     | func
(1 row)
```

## How to see what psql is doing – ECHO_HIDDEN

Let's assume we want to observe the size of the table `t1` - for that, we could construct a query returning table size 
(or just find it somewhere or ask an LLM). But staying inside `psql`, we can just use `\dt+ t1`:

```
nik=# \dt+ t1
                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | t1   | table | postgres | permanent   | heap          | 25 MB |
(1 row)
```

We would like to execute it in a loop, to observe how table size grows. For this, psql supports `\watch` – however, it
won't work with other backslash commands.

Solution – turn on `ECHO_HIDDEN` and see the SQL query behind `\dt+` (alternatively, you can use the option 
`--echo-hidden` when starting `psql`):

```sql
nik=# \set ECHO_HIDDEN 1
nik=#
nik=# \dt+ t1
********* QUERY **********
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'm' THEN 'materialized view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 't' THEN 'TOAST table' WHEN 'f' THEN 'foreign table' WHEN 'p' THEN 'partitioned table' WHEN 'I' THEN 'partitioned index' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner",
  CASE c.relpersistence WHEN 'p' THEN 'permanent' WHEN 't' THEN 'temporary' WHEN 'u' THEN 'unlogged' END as "Persistence",
  am.amname as "Access method",
  pg_catalog.pg_size_pretty(pg_catalog.pg_table_size(c.oid)) as "Size",
  pg_catalog.obj_description(c.oid, 'pg_class') as "Description"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_catalog.pg_am am ON am.oid = c.relam
WHERE c.relkind IN ('r','p','t','s','')
  AND c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
**************************

                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | t1   | table | postgres | permanent   | heap          | 72 MB |
(1 row)
```

Now we have the query, and we can use `\watch 5` to see the result every 5 seconds (while also omitting the fields we
don't need now):

```sql
nik=#   SELECT n.nspname as "Schema",
    c.relname as "Name",
    pg_catalog.pg_size_pretty(pg_catalog.pg_table_size(c.oid)) as "Size"
  FROM pg_catalog.pg_class c
       LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
       LEFT JOIN pg_catalog.pg_am am ON am.oid = c.relam
  WHERE c.relkind IN ('r','p','t','s','')
    AND c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
    AND pg_catalog.pg_table_is_visible(c.oid)
  ORDER BY 1,2 \watch 5
Thu 16 Nov 2023 08:00:10 AM UTC (every 5s)

 Schema | Name | Size
--------+------+-------
 public | t1   | 73 MB
(1 row)

Thu 16 Nov 2023 08:00:15 AM UTC (every 5s)

 Schema | Name | Size
--------+------+-------
 public | t1   | 75 MB
(1 row)

Thu 16 Nov 2023 08:00:20 AM UTC (every 5s)

 Schema | Name | Size
--------+------+-------
 public | t1   | 77 MB
(1 row)
```
