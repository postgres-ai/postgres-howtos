Originally from: [tweet](https://twitter.com/samokhvalov/status/1707854750392455596), [LinkedIn post](...). 

---

# Understanding how sparsely tuples are stored in a table
Today, we'll discuss tuples and their locations in pages – this is quite entry-level material but useful in many cases.

Understanding physical layout of rows in tables may be important in many cases, especially during performance optimization efforts.

## Some terms
- Page / buffer /  block – unit of storage on disk and in Postgres buffer pool (loaded to RAM unchanged), in most cases 8 KiB (check it: `show block_size;`), it holds a portion of a table or index.
- Tuple – physical version of a row in a table.
- Tuple header – metadata about a tuple, including transaction ID, visibility info, and more.
- Transaction ID (same as XID, `tid`, `txid`) – unique  identifier for a transaction in Postgres:
    - It's allocated for modifying transactions. Read-only ones have "virtualxid" to avoid "wasting" XIDs, since they are still 32-bit as of PG16. There is work in progress to switch to 64-bit.(https://commitfest.postgresql.org/43/3594/)
    - You can get a XID allocated for your transactions calling function `pg_current_xact_id()` or, in PG12 and older, `txid_current()`.

Tuple header has interesting "hidden", or "system" columns (docs: https://postgresql.org/docs/current/ddl-system-columns.html):
- `ctid` – a hidden (system) column that represents the physical location of tuple in table, it has the form of two integers `(X, Y)`, where:
    - `X` is page number starting from 0
    - `Y` is sequential number of tuple inside the page starting from 1
- `xmin`, `xmax` – XIDs of transactions that created this row version (tuple), and deleted it (making this tuple "dead")

## ctid
If we need to understand how sparsely some tuples are stored, we can just include `ctid` into the `SELECT` clause of the query. For example, we have the following table and a simple query to it:
```
nik=# \d t1
                      Table "public.t1"
 Column  |       Type       | Collation | Nullable | Default
---------+------------------+-----------+----------+---------
 id      | bigint           |           | not null |
 user_id | double precision |           |          |
Indexes:
    "t1_pkey" PRIMARY KEY, btree (id)
    "t1_user_id_idx" btree (user_id)

nik=# select * from t1 where user_id = 101469;
   id   | user_id
--------+---------
  28414 |  101469
 235702 |  101469
 478876 |  101469
 495042 |  101469
 555593 |  101469
 626491 |  101469
 635785 |  101469
 702725 |  101469
(8 rows)
```

To understand physical locations of these rows, just include `ctid` to the `SELECT` clause of the same query:
```
nik=# select ctid, * from t1 where user_id = 101469;
    ctid    |   id   | user_id
------------+--------+---------
 (153,109)  |  28414 |  101469
 (1274,12)  | 235702 |  101469
 (2588,96)  | 478876 |  101469
 (2675,167) | 495042 |  101469
 (3003,38)  | 555593 |  101469
 (3386,81)  | 626491 |  101469
 (3436,125) | 635785 |  101469
 (3798,95)  | 702725 |  101469
(8 rows)
```

– each row is stored in a different page (`153`, `1274`, etc). This is not the best situation for the performance of this query – a lot of IO is needed.

Now, if we want to see what tuples are present in page 1274, we can use a trick by converting ctid to point (via text since direct conversion is not possible) and extracting the first number of the two – the page number.
```
nik=# select ctid, * from t1 where (ctid::text::point)[0] = 1274 order by ctid limit 15;
   ctid    |   id   | user_id
-----------+--------+---------
 (1274,1)  | 235691 |  225680
 (1274,2)  | 235692 |  617397
 (1274,3)  | 235693 |  233968
 (1274,4)  | 235694 |  714957
 (1274,5)  | 235695 |  742857
 (1274,6)  | 235696 |  837441
 (1274,7)  | 235697 |  745413
 (1274,8)  | 235698 |  752474
 (1274,9)  | 235699 |  102335
 (1274,10) | 235700 |  908715
 (1274,11) | 235701 |  909036
 (1274,12) | 235702 |  101469
 (1274,13) | 235703 |  599451
 (1274,14) | 235704 |  359470
 (1274,15) | 235705 |  196437
(15 rows)

nik=# select count(*) from t1 where (ctid::text::point)[0] = 1274;
 count
-------
   185
(1 row)
```

– overall, there are 185 rows, and our row for `user_id=101469` is also present, at position `12`.

(Note, however, that this will be a very slow query for larger tables since it requires a Seq Scan, and we cannot create an index on `ctid` or other system columns. For queries that are aiming to find particular `ctid` (`...where ctid = '(123, 456)'`), though, performance is going to be good thanks to Tid Scan, see https://pgmustard.com/docs/explain/tid-scan).

## ctid & BUFFERS metrics in execution plans
Confirming that the original query, indeed, involves many buffer operations (also see Day 1 where we talked about importance of `BUFFERS`):
```
nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Index Scan using t1_user_id_idx on t1  (actual time=0.131..0.367 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Buffers: shared hit=11
 Planning Time: 0.159 ms
 Execution Time: 0.383 ms
(5 rows)
```

– 11 buffer hits, it is 88 KiB. This is to retrieve 8 rows. Here is how we can determine the size of those 8 rows:
```
nik=# select sum(pg_column_size(t1.*)) from t1 where user_id = 101469;
 sum
-----
 317
(1 row)
```

Thus, the Postgres executor must handle 88 KiB to return 317 bytes – this is far from optimal. Since we have an Index Scan here, some of those buffer hits are index-related, some – to get data  from heap (table).

## How to improve?

**Option 0.** Don't do anything but understand what's happening. Perhaps, you don't need to make significant improvements, as none of the options discussed below are perfect. Avoid over-optimization. But understand how sparse tuples are located and be ready to double-check it. In some cases, the fact that the target tuples stored too sparsely can be a significant factor for query performance,  leading to timeouts. In this case, do consider the following tactics.

**Option 1.** Maintain tables and indexes in good shape:
- Table bloat control: bloat is regularly analyzed, prevented by well-tuned autovacuum and regularly removed by `pg_repack`.
- Index maintenance: bloat control as well + regular reindexing, because index health declines over time even if autovacuum is well-tuned (btree health degradation rates improved in PG14, but those optimization does not eliminate the need to reindex on regular basis in heavily loaded systems).
- Partitioning: one of benefits of partitioning is improved data locality.

**Option 2.** Use index-only scans instead of index scans. This can be achieved by using mutli-column indexes or covering indexes, to include all the columns needed for our query. For our example:
```
nik=# create index on t1(user_id) include (id);
CREATE INDEX

nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Index Only Scan using t1_user_id_id_idx on t1 (actual time=0.040..0.044 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Heap Fetches: 0
   Buffers: shared hit=4
 Planning Time: 0.179 ms
 Execution Time: 0.072 ms
(6 rows)
```
– 4 buffer hits instead of 11, much better.

**Option 3:** Physically reorganize the table according to the index / column values:
This is physical reorganization of the table. It has two downsides:
- You need to choose which index to use for it – only one index. Therefore, it will help only to specific subset of the queries on the workload and can be useless for other queries
- UPDATEs of the rows will move tuples, decreasing the benefits of `CLUSTER`, so it might be needed to repeat it.

There are two ways to reorganize the table
- SQL command `CLUSTER` (docs: https://postgresql.org/docs/current/sql-cluster.html) – not an online operation, cannot be recommended for live systems that cannot afford maintenance windows
- `pg_repack` (https://github.com/reorg/pg_repack) has option "--order-by=<..>", which allows achieving an effect similar to `CLUSTER` but in an online fashion, without downtime.

For our example:
```
nik=# cluster t1 using t1_user_id_idx;
CLUSTER

And now the query:

nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Index Scan using t1_user_id_idx on t1 (actual time=0.035..0.040 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Buffers: shared hit=4
 Planning Time: 0.141 ms
 Execution Time: 0.070 ms
(5 rows)
```
– also just 4 buffer hits – same as for the approach with covering index and Index-Only Scan. Though, here we have Index Scan.

===

That's it for today – we discussed ctid. Sometime in the future, we'll continue with xmin and xmax and a deep inspection of table/index pages for practical reasons.

[Subscribe](https://twitter.com/samokhvalov), share, like!

Not to forget, I'm mirroring these tips in this GitLab repo: https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/