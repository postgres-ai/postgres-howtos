Originally from: [tweet](https://twitter.com/samokhvalov/status/1719969573318082822), [LinkedIn post]().

---

# How to enable data checksums without downtime

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## Data checksums, the basics

Postgres provides the ability to enable [data checksums](https://postgresql.org/docs/current/checksums.html), which is a
good way to protect from certain types of corruption (not all of them). 

Note WAL has its own checksums, and it's 
[always enabled to verify the integrity of WAL data](https://gitlab.com/postgres/postgres/blob/40d5e5981cc0fa81710dc2399b063a522c36fd68/src/backend/access/transam/xloginsert.c#L896);
in this post we discuss data checksums for table and index pages.

Data checksums are *disabled* by default and can be enabled at cluster initialization time, when executing `initdb` – it
has the option `--data-checksums`.

## Should data checksums be enabled?

Per the [docs](https://postgresql.org/docs/current/app-initdb.html#APP-INITDB-DATA-CHECKSUMS:)

> Enabling checksums may incur a noticeable performance penalty.

However, I strongly recommend enabling data checksums for all clusters. If concerned about the overhead, test it.
Example of a 
[synthetic benchmark](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/44), which
demonstrated a very low (~2%) of CPU load increased. In my opinion, even if this overhead was higher, it would still be
worth having them, considering how important it is to promptly detect storage-level corruption.

## How to check if data checksums are enabled in an existing cluster

There are three ways to check if data checksums are enabled in an existing cluster:

1) With SQL (Postgres has to be online):

   ```
   nik=# show data_checksums;
    data_checksums
   ----------------
    off
   ```

2) Using `pg_controldata` (it doesn't matter if Postgres is online or offline):

   ```bash
   ❯ pg_controldata -D /opt/homebrew/var/postgresql@15 | grep checksum
   Data page checksum version:           0
   ```

   `0` here means data checksums are disabled.

3) Using `pg_checksums` (shipped with Postgres since version 12; in this case, Postgres has to be offline; note that if
   checksums are already enabled, this tool is going to scan the files and check checksums, so you might want to run it
   with the option `--progress` to see the progress)

   ```bash
   ❯ pg_checksums -D /opt/homebrew/var/postgresql@15 --check
   pg_checksums: error: data checksums are not enabled in cluster
   ```

## How to enable data checksums in an existing cluster

Unfortunately, there is no way to enable data checksums in a server which is running. There are two general ways to
enable them:

- cluster re-initialization (dump/restore, logical replication)
- `pg_checksums` (server has to be offline)

In Postgres 12+, there is a tool [pg_checksums](https://postgresql.org/docs/current/app-pgchecksums.html) shipped with
Postgres itself. For older versions (9.3–11), consider using it from [here](https://github.com/credativ/pg_checksums).

When `pg_checksums` is executed, as already mentioned, Postgres has to be shut down.

```bash
❯   brew services stop postgresql@15
Stopping `postgresql@15`... (might take a while)
==> Successfully stopped `postgresql@15` (label: homebrew.mxcl.postgresql@15)

❯ time pg_checksums -D /opt/homebrew/var/postgresql@15 --enable --progress
31035/31035 MB (100%) computed
Checksum operation completed
Files scanned:   3060
Blocks scanned:  3972581
Files written:  1564
Blocks written: 3711369
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
pg_checksums -D /opt/homebrew/var/postgresql@15 --enable --progress  5.19s user 14.23s system 56% cpu 34.293 total
```

– on this MacBook, enabling data checksums was done at speed ~1 GiB/second. Note that `pg_checksums` should be executed
under the same OS user who owns the data directory.

Checking the result:

```bash
❯ pg_controldata -D /opt/homebrew/var/postgresql@15 | grep checksum
Data page checksum version:           1

❯ pg_checksums -D /opt/homebrew/var/postgresql@15 --check --progress
31035/31035 MB (100%) computed
Checksum operation completed
Files scanned:   3060
Blocks scanned:  3972581
Bad checksums:  0
Data checksum version: 1
```

Once it's done we can start Postgres and check again:

```bash
❯ brew services start postgresql@15
==> Successfully started `postgresql@15` (label: homebrew.mxcl.postgresql@15)

❯ psql -Xc 'show data_checksums'
 data_checksums
----------------
 on
(1 row)
```

It is critical to ensure that Postgres is not started while `pg_checksums --enable` is running. Unfortunately,
`pg_checksums` doesn't check it when running (it does it only in the very beginning). There is good trick to avoid
accidental start ([source](https://crunchydata.com/blog/fun-with-pg_checksums)) – move some core file or directory that
Postgres needs for work, temporarily:

```bash
mv $PGDATA/pg_twophase $PGDATA/pg_twophase.DO_NOT_START_THIS_DATABASE
```

... and once `pg_checksums` work is done, move back:

```bash
mv $PGDATA/pg_twophase.DO_NOT_START_THIS_DATABASE $PGDATA/pg_twophase
```

## How to enable data checksums in a multi-node cluster without downtime

Fortunately, if there are replicas in the cluster, we can enable data checksums without downtime – performing a
switchover. The steps are straightforward:

1) Stop a replica (and make sure it won't start till the step 3!)
2) Run `pg_checksum --enable` on it.
3) Start it back and let it fully catch up with the primary.
4) Execute steps 1-3 for all other replicas.
5) Perform a switchover (for minimal downtime, an explicit `CHECKPOINT` is recommended right before switching over; for
   zero downtime, if `pgBouncer` is used, it is recommended to use its `PAUSE/RESUME` capabilities).
6) Apply the steps 1-3 to the ex-primary (not a replica).

Using this approach, a very large clusters can be successfully converted. Before executing this procedure, additional
measures are recommended:

- Test `pg_checksums --enable` on a clone / in a lower environment first, and estimate two values for production:
  duration of `pg_checksums` execution (seconds) and the accumulated lag during it (bytes).
- Plan execution for lower-activity time (e.g., nighttime or weekend, depending on workload profile), to make the lag
  accumulated smaller.

Unfortunately, parallel processing is not yet supported, as of PG16 / 2023, but it is definitely possible that it will
be implemented in the future versions.

As an example, if the speed of conversion is ~1 GiB/second, it means for a 1 TiB cluster, we will need 1024 / 60 ~= 17
minutes. On machines with more powerful disks, it should be much faster.
