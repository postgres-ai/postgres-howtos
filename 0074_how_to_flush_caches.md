Originally from: [tweet](https://twitter.com/samokhvalov/status/1733652860435640705), [LinkedIn post]().

---

# How to flush caches (OS page cache and Postgres buffer pool)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

For experiments, it is important to take into account the state of caches – Postgres buffer pool (size of which is
controlled by `shared_buffers`) and OS page cache. If we decide to start each experiment run with cold caches, we need
to flush them.

## Flushing Postgres buffer pool

To flush Postgres buffer pool, restart Postgres.

To analyze the current state of the buffer pool,
use [pg_buffercache](https://postgresql.org/docs/current/pgbuffercache.html).

## Flushing OS page cache

To flush Linux page cache:

```bash
sync
echo 3 > /proc/sys/vm/drop_caches
```

To see the current state of RAM consumption (in MiB) in Linux:

```bash
free -m
```

On macOS, to flush the page cache:

```bash
sync
sudo purge
```

To see the current state of RAM on macOS:

```bash
vm_stat
```
