Originally from: [tweet](https://twitter.com/samokhvalov/status/1724351012562141612), [LinkedIn post]().

---

# How to use variables in psql scripts

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

`psql` is a native terminal-based client for PostgreSQL. It is very powerful, available on many platforms, is installed
with Postgres (often in a separate package, e.g., `apt install postgresql-client-16` on Ubuntu/Debian).

`psql` supports advanced scripting, and `psql` scripts can be viewed as a superset of Postgres SQL dialect.
For example, it supports commands like `\set`, `\if`, `\watch`. I usually use extension `.psql` for the scripts that are
to be executed by `psql`.

There are two types of variables that can be used in `psql`:

1. Client-side (`psql`'s) – those that are set using `\set` and accessed using **colon-prefixed names**.
2. Server-side parameters (a.k.a. user-defined GUC) – those that can be set and viewed using SQL queries, using keywords
   `SET` and `SHOW` respectively.

## Client-side variables

For a number:

```sql
nik=# \set var1 1.23

nik=# select :var1 as result;
 result
--------
   1.23
(1 row)
```

Note that `\set` is a client-side (`psql`'s) instruction, it doesn't need a semicolon in the end.

For a string value:

```sql
nik=# \set str1 'Hello, world'

nik=# select :'str1' as hi;
      hi
--------------
 Hello, world
(1 row)
```

Note the quite a strange syntax – `:'str1'`. It may require some time to memorize.

Another interesting way to set a `psql`'s client-side variable is to use `\gset` instead of closing semicolon:

```sql
nik=# select now() as ts, current_user as usr \gset

nik=# select :'ts', :'usr';
           ?column?            | ?column?
-------------------------------+----------
 2023-11-14 00:27:53.615579-08 | nik
(1 row)
```

## Server-side variables

The most common way to set a user-defined GUC is `SET`:

```sql
nik=# set myvars.v1 to 1.23;
SET

nik=# show myvars.v1;
 myvars.v1
-----------
 1.23
(1 row)
```

Notes:

- These are SQL queries, ending with a semicolon; they can be executed from other clients as well, not only from `psql`.
- Custom GUC should be accompanied by a "_namespace_" (`set v1 = 1.23;` won't work – un-prefixed parameters are 
  considered as standard GUC, such as `shared_buffers`).
- Working with strings is straightforward (`set myvars.v1 to 'hello';`).

Values defined by using `SET` do not persist – they are present only during the ongoing session (or, if `SET LOCAL` is
used, only during the current transaction). For persistence, use either of these approaches.

1) Cluster-wide:
    ```sql
    nik=# alter system set myvars.v1 to 2;
    ALTER SYSTEM
    
    nik=# select pg_reload_conf();
     pg_reload_conf
     ----------------
      t
     (1 row)
     
     nik=# \c
     You are now connected to database "nik" as user "nik".
 
     nik=# show myvars.v1;
      myvars.v1
     -----------
      2
    (1 row)
    ```

   Notice the config reload using `pg_reload_conf()` and a reconnection.

2) At database level:

    ```sql
    nik=# alter database nik set myvars.v1 to 3;
    ALTER DATABASE
   
    nik=# \c
    You are now connected to database "nik" as user "nik".
   
    nik=# show myvars.v1;
     myvars.v1
    -----------
     3
    (1 row)
    ```

3) At user level:

    ```sql
    nik=# alter user nik set myvars.v1 to 4;
    ALTER ROLE
   
    nik=# \c
    You are now connected to database "nik" as user "nik".
   
    nik=# show myvars.v1;
     myvars.v1
    -----------
     4
    (1 row)
    ```

## Server-side variables – how to integrate with SQL

This `SET`/`SHOW` syntax is very common. However, it is often inconvenient because neither `SET` nor `SHOW` can be
integrated to other SQL queries such as `SELECT`. To solve this, use alternative methods to set and
access – `set_config(...)`
and `current_setting(...)` ([docs](https://postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET)).

Instead of `SET`, use `set_config(...)`:

```sql
nik=# select set_config('myvars.v1', '5', false);
 set_config
------------
 5
(1 row)

nik=# show myvars.v1;
 myvars.v1
-----------
 5
(1 row)
```

Note that the value can be text-only – so for numbers and other data types, subsequent conversion may be needed.

And instead of `SHOW`, use `current_setting(...)`:

```sql
nik=# select set_config('myvars.v1', '6', false);
 set_config
------------
 6
(1 row)

nik=# select current_setting('myvars.v1', true)::int;
 current_setting
-----------------
               6
(1 row)
```

## Variables in anonymous DO blocks

💡👉 **Idea from
[passing parameters from command line to DO statement](https://postgres.cz/wiki/PostgreSQL_SQL_Tricks#Passing_parameters_from_command_line_to_DO_statement)**.

Anonymous `DO` blocks do not support client-side variables, so we need to pass them to the server-side first:

```sql
nik=# \set loops 5

nik=# select set_config('myvars.loops', (:loops)::text, false);
 set_config
------------
 5
(1 row)

nik=# do $$
begin
  for i in 1..current_setting('myvars.loops', true)::int loop
    raise notice 'Iteration %', i;
  end loop;
end $$;
NOTICE:  Iteration 1
NOTICE:  Iteration 2
NOTICE:  Iteration 3
NOTICE:  Iteration 4
NOTICE:  Iteration 5
DO
```

## Passing variables to .psql scripts

Consider that we have a script named `largest_tables.psql`:

```sql
❯ cat largest_tables.psql

  select
    relname,
    pg_total_relation_size(oid::regclass),
    pg_size_pretty(pg_total_relation_size(oid::regclass))
  from pg_class
  order by pg_total_relation_size(oid::regclass) desc
  limit :limit;
```

Now, we can call it dynamically by setting the value for client-side variable `limit`:

```sql
❯ psql -X -f largest_tables.psql -v limit=2
     relname      | pg_total_relation_size | pg_size_pretty
------------------+------------------------+----------------
 pgbench_accounts |              164732928 | 157 MB
 t13              |               36741120 | 35 MB
(2 rows)

❯ PGAPPNAME=mypsql psql -X \
  -f largest_tables.psql \
  -v limit=3 \
  --csv
relname,pg_total_relation_size,pg_size_pretty
pgbench_accounts,164732928,157 MB
t13,36741120,35 MB
tttttt,36741120,35 MB
```
