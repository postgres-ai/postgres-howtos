Originally from: [tweet](https://twitter.com/samokhvalov/status/1720734029207732456), [LinkedIn post]().

---

# How to break a database, Part 1: How to corrupt

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Sometimes, you might want to damage a database – for educational purposes, to simulate failures, learn how to deal with
them, to test mitigation procedures.

Let's discuss some ways to break things.

<span style="padding: 1ex; background-color: yellow">
⚠️ Don't do it in production unless you're a chaos engineer ⚠️
</span>

## Corruption

There are many types of corruption and there are very simple ways to get a corrupted database, for example:

👉 **Modifying system catalogs directly:**

```sql
nik=# create table t1(id int8 primary key, val text);
CREATE TABLE

nik=# delete from pg_attribute where attrelid = 't1'::regclass and attname = 'val';
DELETE 1

nik=# table t1;
ERROR:  pg_attribute catalog is missing 1 attribute(s) for relation OID 107006
LINE 1: table t1;
          ^
```

More ways can be found in this article:
[How to corrupt your PostgreSQL database](https://cybertec-postgresql.com/en/how-to-corrupt-your-postgresql-database/).
A couple of interesting methods from there:

- `fsync=off` + `kill -9` to Postgres (or `pg_ctl stop -m immediate`)
- `kill -9` + `pg_resetwal -f`

One useful method is to use `dd` to write to a data file directly. This can be used to simulate a corruption that can be
detected by checksum verification
([Day 37: How to enable data checksums without downtime](0037_how_to_enable_data_checksums_without_downtime.md)). This
is also demonstrated in this article:
[pg_healer: repairing Postgres problems automatically](https://endpointdev.com/blog/2016/09/pghealer-repairing-postgres-problems/).

First, create a table and see where its data file is located:

```sql
nik=# show data_checksums;
 data_checksums
----------------
 on
(1 row)

nik=# create table t1 as select i from generate_series(1, 10000) i;
SELECT 10000

nik=# select count(*) from t1;
 count
-------
 10000
(1 row)

nik=# select format('%s/%s',
  current_setting('data_directory'),
  pg_relation_filepath('t1'));
                      format
---------------------------------------------------
 /opt/homebrew/var/postgresql@15/base/16384/123388
(1 row)
```

Now, let's write some garbage to this file directly, using `dd` (note that here we use a macOS version, where `dd` has
the option `oseek` – on Linux, it's `seek_bytes`), and then restart Postgres to make sure the table is not present in
the buffer pool anymore:

```bash
❯ echo -n "BOOoo" \
  | dd conv=notrunc bs=1 \
    oseek=4000 count=1 \
    of=/opt/homebrew/var/postgresql@15/base/16384/123388
 1+0 records in
 1+0 records out
 1 bytes transferred in 0.000240 secs (4167 bytes/sec)

❯ brew services stop postgresql@15
 Stopping `postgresql@15`... (might take a while)
 ==> Successfully stopped `postgresql@15` (label: homebrew.mxcl.postgresql@15)

❯ brew services start postgresql@15
 ==> Successfully started `postgresql@15` (label: homebrew.mxcl.postgresql@15)
```

Successfully corrupted – the data checksums mechanism complains about it:

```sql
nik=# table t1;
WARNING:  page verification failed, calculated checksum 52379 but expected 35499
ERROR:  invalid page in block 0 of relation base/16384/123388
```

**🔜 To be continued ...**
