Originally from: [tweet](https://twitter.com/samokhvalov/status/1717044844227637530), [LinkedIn post]().

---

## How to work with arrays, part 2

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Part 1 can be found [here](0028_how_to_work_with_arrays_part_1.md).

## How to search

One of the most interesting operators for array search is `<@` – "is contained in":

```
nik=# select array[2, 1] <@ array[1, 3, 2];
 ?column?
----------
 t
(1 row)
```

When you need to check if some scalar value is present in an array, just create a single-element array and apply `<@` to
two arrays. For example, checking if "2" is contained in the analyzed array:

```
nik=# select array[2] <@ array[1, 3, 2];
 ?column?
----------
 t
(1 row)
```

You can build an index to speed up search using `<@` - let's create a GIN index and compare the plans for a table with
a million rows:

```
nik=# create table t1 (val int8[]);
CREATE TABLE

nik=# insert into t1
select array(
  select round(random() * 1000 + i)
  from generate_series(1, 10)
  limit (1 + random() * 10)::int
)
from generate_series(1, 1000000) as i;
INSERT 0 1000000

nik=# select * from t1 limit 3;
          val
-----------------------
 {390,13,405,333,358,592,756,677}
 {463,677,585,191,425,143}
 {825,918,303,602}
(3 rows)

nik=# vacuum analyze t1;
VACUUM
```

– we created a single-column table with 1M rows containing `int8[]` arrays with various numbers.

Search all rows containing "123" in the array value:

```
nik=# explain (analyze, buffers) select * from t1 where array[123]::int8[] <@ val;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
  Gather  (cost=1000.00..18950.33 rows=5000 width=68) (actual time=0.212..100.572 rows=2 loops=1)
    Workers Planned: 2
    Workers Launched: 2
    Buffers: shared hit=12554
    ->  Parallel Seq Scan on t1  (cost=0.00..17450.33 rows=2083 width=68) (actual time=61.293..94.212 rows=1 loops=3)
          Filter: ('{123}'::bigint[] <@ val)
          Rows Removed by Filter: 333333
          Buffers: shared hit=12554
  Planning:
    Buffers: shared hit=6
  Planning Time: 0.316 ms
  Execution Time: 100.586 ms
(12 rows)
```

– 12554 buffer hits, or 12554 * 8 / 1024 ~= 98 MiB. Just to find those 2 rows with an array value containing "123" -
notice "rows=2". It's not efficient; we have a Seq Scan here.

Now, with a GIN index:

```
nik=# create index on t1 using gin(val);
CREATE INDEX

nik=# explain (analyze, buffers) select * from t1 where array[123]::int8[] <@ val;
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
  Bitmap Heap Scan on t1  (cost=44.75..4260.25 rows=5000 width=68) (actual time=0.021..0.022 rows=2 loops=1)
    Recheck Cond: ('{123}'::bigint[] <@ val)
    Heap Blocks: exact=1
    Buffers: shared hit=5
    ->  Bitmap Index Scan on t1_val_idx  (cost=0.00..43.50 rows=5000 width=0) (actual time=0.016..0.016 rows=2 loops=1)
          Index Cond: (val @> '{123}'::bigint[])
          Buffers: shared hit=4
  Planning:
    Buffers: shared hit=16
  Planning Time: 0.412 ms
  Execution Time: 0.068 ms
(11 rows)
```

– no more Seq Scan, and we have as few as 5 buffer hits, or 40 KiB to find 2 rows we need. This explains why the
execution time went from ~100ms down to ~0.07ms, this is ~1400x faster.

More operators in [official docs](https://postgresql.org/docs/current/functions-array.html#FUNCTIONS-ARRAY).
