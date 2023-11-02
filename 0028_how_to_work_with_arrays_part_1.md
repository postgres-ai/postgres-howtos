Originally from: [tweet](https://twitter.com/samokhvalov/status/1716730586969280809), [LinkedIn post]().

---

## How to work with arrays, part 1

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Let's talk about arrays today: [Docs](https://postgresql.org/docs/current/arrays.html).

Arrays in Postgres are powerful and existed long before JSON was added. It is worth knowing how to use them to achieve a
new level of expressive power (and not just manipulations – you can store them, tables can have columns of array type,
even multi-dimensional ones).

If you need to choose just one fact to remember about Postgres arrays, here it is: Postgres is not like others, so array
indexing starts with 1, not 0. And don't expect it to complain about an attempt to use 0:

```
nik=# \pset null '(null)'
Null display is "(null)".

nik=# select (array['one', 'two'])[0];
 array
--------
 (null)
(1 row)

nik=# select (array['one', 'two'])[1];
 array
-------
 one
(1 row)
```

## Creating arrays

### Method 1: define array value as a string and converting it to array type

```
nik=# select '{"one", "two"}'::text[];
   text
-----------
 {one,two}
(1 row)
```

### Method 2:  using `array[...]`

```
nik=# select array['one', 'two'];
   array
-----------
 {one,two}
(1 row)
```

### Method 3: using `select array( <subquery> );`

```
nik=# select array( values('one'), ('two') );
   array
-----------
 {one,two}
(1 row)
```

This method helps pack many 1-column rows into an array value:

```
nik=# select array(select pid from pg_stat_activity);
                    array
---------------------------------------------
 {54757,54759,26135,54751,54758,54750,54756}
(1 row)
```

Interestingly, you can "pack" the whole table into a single value - more precisely, a single-column, single-row table:

```
nik=# select n from pg_namespace as n \gx
-[ RECORD 1 ]-------------------------------------------------------------------------
n | (99,pg_toast,10,)
-[ RECORD 2 ]-------------------------------------------------------------------------
n | (11,pg_catalog,10,"{nik=UC/nik,=U/nik}")
-[ RECORD 3 ]-------------------------------------------------------------------------
n | (2200,public,6171,"{pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}")
-[ RECORD 4 ]-------------------------------------------------------------------------
n | (13471,information_schema,10,"{nik=UC/nik,=U/nik}")
-[ RECORD 5 ]-------------------------------------------------------------------------
n | (41022,pgss_pgsa,10,)

nik=# select array(select n from pg_namespace as n) \gx
-[ RECORD 1 ]------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
array | {"(99,pg_toast,10,)","(11,pg_catalog,10,\"{nik=UC/nik,=U/nik}\")","(2200,public,6171,\"{pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}\")","(13471,information_schema,10,\"{nik=UC/nik,=U/nik}\")","(41022,pgss_pgsa,10,)"}
```

This value retains column names, and you can unpack it back, using `unnest(..)` and `(t).*`:

```
nik=# with packed(val) as (
  select array(select n from pg_namespace as n)
)
select unnest(val)
from packed;
                                       unnest
------------------------------------------------------------------------------------
 (99,pg_toast,10,)
 (11,pg_catalog,10,"{nik=UC/nik,=U/nik}")
 (2200,public,6171,"{pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}")
 (13471,information_schema,10,"{nik=UC/nik,=U/nik}")
 (41022,pgss_pgsa,10,)
(5 rows)

nik=# with packed(val) as (
  select array(select n from pg_namespace as n)
)
select (unnest(val)).*
from packed;
  oid  |      nspname       | nspowner |                            nspacl
-------+--------------------+----------+---------------------------------------------------------------
    99 | pg_toast           |       10 | (null)
    11 | pg_catalog         |       10 | {nik=UC/nik,=U/nik}
  2200 | public             |     6171 | {pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}
 13471 | information_schema |       10 | {nik=UC/nik,=U/nik}
 41022 | pgss_pgsa          |       10 | (null)
(5 rows)
```

Magic! This can help you write very complex CTEs. (It is recommended to always comment each step though – Postgres
supports SQL-standard `--` commenting, as well as `/* ... */`.)

### Method 4: aggregation functions

```
nik=# select array_agg(i) from generate_series(1, 3) as i;
 array_agg
-----------
 {1,2,3}
(1 row)
```

This method can also be used to pack all single-column rows into one array value:

```
nik=# select array_agg(pid) from pg_stat_activity;
                  array_agg
---------------------------------------------
 {54757,54759,26135,54751,54758,54750,54756}
(1 row)
```

## How to access array elements

Accessing array members is pretty straightforward:

```
nik=# select (array_agg(pid))[1] from pg_stat_activity;
 array_agg
-----------
     54757
(1 row)
```

Again, don't forget that indexes start with 1, and accessing a non-existent member won't produce an error, it will
return `NULL`:

```
nik=# select (array_agg(pid))[100000] from pg_stat_activity;
 array_agg
-----------
    (null)
(1 row)
```

You can use `arr[N:M]` to get only a "slice" of array. And you can omit one of the numbers – for example, to keep only
the first 3 members, use `arr[:3]`:

```
nik=# select (array_agg(pid))[:3] from pg_stat_activity;
 array_agg
---------------------
 {54757,54759,26135}
(1 row)
```

// To be continued
