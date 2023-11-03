Originally from: [tweet](https://twitter.com/samokhvalov/status/1713795383183474867), [LinkedIn post](...).

---

# How to use pg_restore

Today – a few tips on using `pg_restore` to restore databases (or only parts of them) from dumps.
Docs: https://postgresql.org/docs/current/app-pgrestore.html.

## Parallelization vs. single table limitation

When dealing with dumps created in the "directory" format (`-Fd`), it is possible to speed up restoring, using the
`-j` (`--jobs`) option, running multiple pg_restore workers. However, it won't be helpful if there is one or a few
tables that are large – parallelization of a single table dump or restoration is not supported. We discussed it on
[Day 8: pg_dump howto](0008_how_to_speed_up_pg_dump.md).

For example, if you create a standard `pgbench` database (say, `pgbench -i -s1000`), you'll find out that
parallelization of both dump and restore is not really helpful because the majority of the database data is stored in a
single table, `pgbench_accounts`. However, if you use partitioning (supported in
PG13+, `pgbench -i -s1000 --partitions=16`), you'll find out that parallelization helps to speed up both dump and
restore steps.

## Atomic restore

By default, pg_restore won't stop on errors. This might be surprising since we got used to more `strict` behavior when
dealing with Postgres. And this also might lead to situations when database restored only partially, but this remained
unnoticed. To switch to the `strict` mode, use `-e` (`--exit-on-error`). It can be also helpful to wrap the restoration
process into a single transaction, using option `-1` (`--single-transaction`).

## Details

To see the progress and details of restoration, use `--verbose`.

## Schema vs. data split

You can look at your dumps at two different angles, both offering a way to structure the dump at high level.

First, you can distinguish schema and data – and use options:

* `-s` (`--schema-only`) – restore only schema
* `-a` (`--data-only`) – restore only data (of course, schema must already exist).

Interestingly, for a dump, this split – "schema + data" – is not the most efficient in terms of restoration time and the
quality of result: indexes are a part of the schema, but if you create them first, and only then load data, the loading
will take longer, and indexes will end up having worse shape than if build after the data load.

Therefore, there is a second way to look at the dump structure that corresponds to the regular order of the full restore
process:

1. "pre-data" – schema definitions except indexes, constraints, triggers, and rules
2. "data" – well, table data
3. "post-data" – indexes, constraints, triggers, and and rules

The option `--section` allows you to run only one of these three steps of the restore process. This may be helpful if
you want to perform restoration as normal, but do some additional actions between the steps or, say, use different
parallelization levels (`-j`) for different steps.

## Fine-grained control

There is a way to achieve even more fine-grained control over the restoration process.

For dumps created using the "directory" format (`-Fd`), you can control what to restore. For this, two convenient
options exist: `-l` to list the content, and `-L` to filter what you need. For example, consider this:

```bash
pgbench -i -s100 test --partitions 16
pg_dump -f dump1 -j8 -Fd test
```

Now, we can quickly list the content of the dump using `-l` (`--list`):

```bash
❯ pg_restore -l dump1
;
; Archive created at 2023-10-15 22:00:43 PDT
; dbname: test
; TOC Entries: 94
; Compression: -1
; Dump Version: 1.14-0
; Format: DIRECTORY
; Integer: 4 bytes
; Offset: 8 bytes
; Dumped from database version: 15.4 (Homebrew)
; Dumped by pg_dump version: 15.4 (Homebrew)
;
;
; Selected TOC Entries:
;
216; 1259 82596 TABLE public pgbench_accounts nik
218; 1259 82602 TABLE public pgbench_accounts_1 nik
227; 1259 82629 TABLE public pgbench_accounts_10 nik
228; 1259 82632 TABLE public pgbench_accounts_11 nik
229; 1259 82635 TABLE public pgbench_accounts_12 nik
...
3597; 0 0 INDEX ATTACH public pgbench_accounts_8_pkey nik
3598; 0 0 INDEX ATTACH public pgbench_accounts_9_pkey nik
```

For example, if we want to restore everything except indexes, we first need to prepare a "list" file containing all
objects in the dump without constraints and indexes:

```bash
❯ pg_restore -l dump1 \
| grep -v INDEX \
 | grep -v CONSTRAINT \
 > no_indexes.list
```

And now use this list in the `-L` (`--use-list`) option:

```bash
❯ psql -c 'create database restored'
CREATE DATABASE
❯ pg_restore -j8 -L no_indexes.list --dbname=restored dump1
```

As we can see, the restored tables don't have PKs or indexes:

```bash
❯ psql restored -c '\d pgbench_accounts_1'
           Table "public.pgbench_accounts_1"
Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
aid      | integer       |           | not null |
bid      | integer       |           |          |
abalance | integer       |           |          |
filler   | character(84) |           |          |
Partition of: pgbench_accounts FOR VALUES FROM (MINVALUE) TO (625001)
```

## Permissions, ownership, etc.

In some cases, when copying or moving schema and data between different environments or clusters, the following options
can help a lot (the names are self-explanatory):

* `--no-owner`
* `--no-privileges`
* `--no-security-labels`
* `--no-publications`
* `--no-subscriptions`
* `--no-tablespaces`

However, if RLS (row-level security) is used in the original database, there isn't a single option to skip restoration
of the `CREATE POLICY` queries the dump contains. In this case, we need to use the previous method for the fine-grained
control to remove the `CREATE POLICY` commands.

So, a possible recipe for restoring from a dump when moving schema&data between different environments may look like
this:

```bash
pg_restore --list /path/to/dump \
    | grep -v  POLICY \
  > dump-no-policies.list

pg_restore \
  --jobs=8 \
  --dbname=${DBNAME}
  --use-list=./dump-no-policies.list \
  --no-owner \
  --no-privileges \
  --no-security-labels \
  --no-publications \
  --no-subscriptions \
  --no-tablespaces \
  --verbose \
  --exit-on-error \
  /path/to/dump
```

## Postscript

Completely forgot (what many people forgot all the time too – it should be default behavior of `pg_restore`, but it's
not):

After running `pg_restore`, don't forget:

- to gather statistics (unless you want to wait until autovacuum does it) – `ANALYZE`
- build visibility maps – `VACUUM`

Hence: `VACUUM ANALYZE`. Can also be parallelized: `vacuumdb --jobs=$N`.
