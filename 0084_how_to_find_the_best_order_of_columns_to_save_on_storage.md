Originally from: [tweet](https://twitter.com/samokhvalov/status/1737733195511312651), [LinkedIn post]().

---

# How to find the best order of columns to save on storage ("Column Tetris")

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Does the column order matter in Postgres, storage-wise?

The answer is **yes**. Let's consider an example (as usual, my advice is NOT to use int4 PKs, but here it is done for
educational purposes):

```sql
create table t(
  id int4 primary key,
  created_at timestamptz,
  is_public boolean,
  modified_at timestamptz,
  verified boolean,
  published_at timestamptz,
  score int2
);

insert into t
select
  i,
  clock_timestamp(),
  true,
  clock_timestamp(),
  true,
  clock_timestamp(),
  0
from generate_series(1, 1000000) as i;

vacuum analyze t;
```

Checking the size:

```
nik=# \dt+ t
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t    | table | postgres | 81 MB |
(1 row)
```

Now, let's use the report `p1` from [postgres_dba](https://github.com/NikolayS/postgres_dba) (assuming it's installed):

```
:dba

Type your choice and press <Enter>:
p1
 Table | Table Size | Comment |    Wasted *     |  Suggested Columns Reorder
-------+------------+---------+-----------------+-----------------------------
 t     | 81 MB      |         | ~23 MB (28.40%) | is_public, score, verified +
       |            |         |                 | id, created_at, modified_at+
       |            |         |                 | published_at
(1 row)
```

-- the report claims we can save `~28%` of disk space just changing the column order. Note that this is an estimate.

Let's check the optimized order:

```sql
drop table t;

create table t(
  is_public boolean,
  verified boolean,
  score int2,
  id int4 primary key,
  created_at timestamptz,
  modified_at timestamptz,
  published_at timestamptz
);

insert into t
select
  true,
  true,
  0::int2,
  i::int4,
  clock_timestamp(),
  clock_timestamp(),
  clock_timestamp()
from generate_series(1, 1000000) as i;

vacuum analyze t;
```

Checking the size:

```
nik=# \dt+ t
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t    | table | postgres | 57 MB |
(1 row)
```

– we saved `~30%`, very close to what was expected (57 / 81 ~= 0.7037).

postgres_dba's report p1 doesn't show potential savings anymore:

```
  Type your choice and press <Enter>:
  p1
   Table | Table Size | Comment | Wasted * | Suggested Columns Reorder
  -------+------------+---------+----------+---------------------------
   t     | 57 MB      |         |          |
  (1 row)
```

Why did we save?

In Postgres, the storage system may add alignment padding to column values. If a data type's natural alignment
requirement is more than the size of the value, Postgres may pad the value with zeros up to the alignment boundary. For
instance, when a column has a value with a size less than 8 bytes followed by a value requiring 8-byte alignment,
Postgres pads the first value to align on an 8-byte boundary. This helps ensure that the values in memory are aligned
with CPU word boundaries for the particular hardware architecture, which can lead to performance improvements.

As an example, a row (`int4`, `timestamptz`) occupies 16 bytes:

- 4 bytes for `int4`
- 4 zeroes to align to 8-bytes
- 8 bytes for `timestamptz`

Some people prefer an inverted approach: first we start with 16- and 8-byte columns, then proceed to smaller ones. In
any case, it makes sense to put columns having `VARLENA` types (`text`, `varchar`, `json`, `jsonb`, array) in the end.

Articles to read on this topic:

- [StackOverflow answer by Erwin Brandstetter](https://stackoverflow.com/questions/2966524/calculating-and-saving-space-in-postgresql/7431468#7431468)
- [Ordering Table Columns in PostgreSQL (GitLab)](https://docs.gitlab.com/ee/development/database/ordering_table_columns.html)
- Docs: [Table Row Layout](https://postgresql.org/docs/current/storage-page-layout.html#STORAGE-TUPLE-LAYOUT)
