Originally from: [tweet](https://twitter.com/samokhvalov/status/1737747838573150476), [LinkedIn post]().

---

# How to quickly check data type and storage size of a value

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Here is how you can quickly check data type and size of a value, not looking in documentation.

## How to check data type for a value

Use `pg_typeof(...)`:

```sql
nik=# select pg_typeof(1);
 pg_typeof
-----------
 integer
(1 row)

nik=# select pg_typeof(1::int8);
 pg_typeof
-----------
 bigint
(1 row)
```

## How to check storage size for a value

Use `pg_column_size(...)` – even without actual column:

```sql
nik=# select pg_column_size(1);
 pg_column_size
----------------
              4
(1 row)

nik=# select pg_column_size(1::int8);
 pg_column_size
----------------
              8
(1 row)
```

👉 `int4` (aka `int` or `integer`) occupies 4 bytes, as expected, while `int8` (`bigint`) – 8.

And `VARLENA` types such as `varchar`, `text`, `json`, `jsonb`, arrays have variable length (hence the name), and 
additional header, e.g.:
    
```sql
nik=# select pg_column_size('ok'::text);
 pg_column_size
----------------
              6
(1 row)
```

👉 a 4-byte `VARLENA` header (for inline storage,
see [struct varlena header](https://github.com/postgres/postgres/blob/c161ab74f76af8e0f3c6b349438525ad9575683b/src/include/c.h#L661-L681))
and 2 bytes for data.

Boolean example:
    
```
nik=# select pg_column_size(true), pg_column_size(false);
 pg_column_size | pg_column_size
----------------+----------------
              1 |              1
(1 row)
```

Remembering the previous howto, [Column Tetris](0084_how_to_find_the_best_order_of_columns_to_save_on_storage.md), here we
can conclude that not only we need 1 byte to store a bit (8x space), it becomes 8 bytes if we create a table 
(`c1 boolean`, `c2 int8`), due to alignment padding – meaning that it's already 64 bits! So, in such "unfortunate" case, those
who store 'true' as text, don't lose anything at all:

```sql
nik=# select pg_column_size('true'::text);
 pg_column_size
----------------
              8
(1 row)
```

👉 it is also 8 bytes (4 bytes `VARLENA` header, and 4 bytes actual value). But if it's `false` as text, then – with
alignment padding (if the next column is 8- or 16-byte), it can quickly become even worse:

```sql
nik=# select pg_column_size('false'::text);
 pg_column_size
----------------
              9
(1 row)
```

👉 these 9 bytes are padded with zeroes to 16, when alignment padding is needed. Don't store true/false as text :) And
for very optimized storage of multiple boolean "flag" values, consider using a single integer, and "packing" boolean
values inside it, then bit operators `<<`, `~`, `|`, `&` to encode and decode
values (docs: [Bit String Functions and Operators](https://postgresql.org/docs/current/functions-bitstring.html)) – 
doing so, you can "pack" 64 booleans inside a single `int8` value.

A couple of more examples:

```sql
nik=# select pg_column_size(interval '1s');
pg_column_size
---------------
             8
(1 row)

nik=# select pg_column_size(interval '1s');
 pg_column_size
----------------
             16
(1 row)
```

## How to check storage size for a row

And a couple of more examples – how to check the size of a row:

```sql
nik=# select pg_column_size(row(true, now()));
 pg_column_size
----------------
             40
(1 row)
```

👉 a 24-byte tuple header (23 bytes padded with 1 zero), then 1-byte boolean (padded with 7 zeroes), and then 8-byte
`timestamptz` value. Total: 24 + 1+7 + 8 = 40.

```sql
nik=# select pg_column_size(row(1, 2));
 pg_column_size
----------------
           32
(1 row)
```

👉 a 24-byte tuple header, then two 4-byte integers, no padding needed, total is 24 + 4 + 4 = 32.

## No need to remember exact function names

When working in psql, there is no need to remember function names – use `\df+` to search function name:

```sql
nik=# \df *pg_*type*
                                           List of functions
   Schema   |                   Name                    | Result data type | Argument data types | Type
------------+-------------------------------------------+------------------+---------------------+------
 pg_catalog | binary_upgrade_set_next_array_pg_type_oid | void             | oid                 | func
 pg_catalog | binary_upgrade_set_next_pg_type_oid       | void             | oid                 | func
 pg_catalog | binary_upgrade_set_next_toast_pg_type_oid | void             | oid                 | func
 pg_catalog | pg_stat_get_backend_wait_event_type       | text             | integer             | func
 pg_catalog | pg_type_is_visible                        | boolean          | oid                 | func
 pg_catalog | pg_typeof                                 | regtype          | "any"               | func
(6 rows)
```
