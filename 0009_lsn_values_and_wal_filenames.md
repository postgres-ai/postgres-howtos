Originally from: [tweet](https://twitter.com/samokhvalov/status/1709818510787162429), [LinkedIn post](...). 

---

# How to understand LSN values and WAL filenames

<img src="files/0009_cover.jpg" width="600" />

## How to read LSN values
LSN – Log Sequence Number, a pointer to a location in the Write-Ahead Log (WAL). Understanding it and how to work with it is important for dealing with physical and logical replication, backups, recovery. Postgres docs:
- [WAL Internals](https://postgresql.org/docs/current/wal-internals.html)
- [pg_lsn Type](https://postgresql.org/docs/current/datatype-pg-lsn.html)

LSN is a 8-byte (32-bit) value ([source code](https://gitlab.com/postgres/postgres/blob/4f2994647ff1e1209829a0085ca0c8d237dbbbb4/src/include/access/xlogdefs.h#L17)). It can be represented in the form of `A/B` (more specifically, `A/BBbbbbbb`, see below), where both `A` and `B` are 4-byte values. For example:
```
nik=# select pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 5D/257E19B0
(1 row)
```

- `5D` here is higher 4-byte (32-bit) section of LSN
- `257E19B0` can, in its turn, be split to two parts as well:
    - `25` – lower 4-byte section of LSN (more specifically, only the highest 1 byte of of that 4-byte section)
    - `7E19B0` – offset in WAL (which is `16 MiB` by default; in some cases, it's changed – e.g., RDS changed it to `64 MiB`)

Interesting that LSN values can be compared, and even subtracted one from each other – assuming we use the pg_lsn data type. The result will be in bytes:
```
nik=# select pg_lsn '5D/257D6780' - '5D/251A8118';
 ?column?
----------
  6481512
(1 row)
 nik=# select pg_size_pretty(pg_lsn '5D/257D6780' - '5D/251A8118');
 pg_size_pretty
----------------
 6330 kB
(1 row)
```

This also means that we can get the integer value of LSN just seeing how far we went from the "point 0" – value `0/0`:
```
nik=# select pg_lsn '5D/257D6780' - '0/0';
   ?column?
--------------
 400060934016
(1 row)
```

## How to read WAL filenames
Now let's see how the LSN values correspond to WAL file names (files located in `$PGDATA/pg_wal`). We can get WAL file name for any given LSN using function `pg_walfile_name()`:
```
nik=# select pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
 pg_current_wal_lsn |     pg_walfile_name
--------------------+--------------------------
 5D/257E19B0        | 000000010000005D00000025
(1 row)
```

Here `000000010000005D00000025` is WAL filename, it consists of three 4-byte (32-bit) words:
1. `00000001` – timeline ID (TimeLineID), a sequential "history number" that starts with 1 when Postgres cluster is initialized. It "identifies different database histories to prevent confusion after restoring a prior state of a database installation" ([source code](https://gitlab.com/postgres/postgres/blob/4f2994647ff1e1209829a0085ca0c8d237dbbbb4/src/include/access/xlogdefs.h#L50)).
2. `0000005D` – higher 4-byte section of sequence number.
3. `00000025` – can be viewed as two parts:
    - `000000` – 6 leading zeroes,
    - `25` – the highest byte of the lower section of sequence number.

Important to remember: the third 4-byte word in the WAL filename has 6 leading zeroes – often, this leads to confusion and mistakes when comparing two WAL filenames to understand what LSNs to expect inside them.

Useful illustration (from [this post](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames)):
```
LSN:                5D/      25   7E19B0
WAL: 00000001 0000005D 00000025
```

This can be very helpful if you need to work with LSN values or WAL filenames or with both of them, quickly navigating or comparing their values to understand the distance between them. Some examples when it can be useful to understand:
1. How much bytes the server generates per day
2. How much has passed since replication slot has been created
3. What's the distance between two backups
4. How much of WAL data needs to be replayed to reach consistency point

## Good blog posts worth reading besides the official docs:
- [Postgres 9.4 feature highlight - LSN datatype](https://paquier.xyz/postgresql-2/postgres-9-4-feature-highlight-lsn-datatype/)
- [Postgres WAL Files and Sequence Numbers](https://crunchydata.com/blog/postgres-wal-files-and-sequuence-numbers)
- [WAL, LSN, and File Names](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames)

---

Thanks for reading! As usual: please share it with your colleagues and friends who work with #PostgreSQL.