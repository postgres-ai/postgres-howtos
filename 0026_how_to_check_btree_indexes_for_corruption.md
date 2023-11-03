Originally from: [tweet](https://twitter.com/samokhvalov/status/1716001897839272288), [LinkedIn post]().

---

# How to check btree indexes for corruption

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

This snippet checks all btree indexes using `$N` parallel workers (parallelization is generally recommended):

```bash
pg_amcheck \
  --install-missing \
  --verbose \
  --progress \
  --jobs=$N \
  --heapallindexed \
  2>&1 \
  | ts \
  | tee -a amcheck_(date "+%F-%H-%M").log
```

Docs:

- [amcheck (extension)](https://postgresql.org/docs/current/amcheck.html)
- [pg_amcheck (CLI tool)](https://postgresql.org/docs/current/app-pgamcheck.html)

Notes:

1. This type of check acquires only `AccessShareLock` on indexes – hence, DML is not blocked. However, in general, this
   kind of check produces a significant workload since it needs to read all indexes and corresponding tuples in headers.

2. The option `--heapallindexed` here is optional but highly recommended. Without it, the check usually takes much less
   time, but in this case only "light" index check is performed (not involving "heap" – it doesn't follow references to
   tuples in tables, checking indexes only).

3. There is another useful option not used here, `--parent-check`, which provides more comprehensive check of index
   structure – it checks parent/child relationships in the index. This is a very useful for "deep" test of indexes.
   However, it is slow and, unfortunately, requires ShareLock locks to be acquired. It means, while working in this
   mode, such checks are blocking modifications (`INSERT`/`UPDATE`/`DELETE`). So, `--parent-check` can only be used
   either on clones not receiving traffic or during maintenance window.

4. Checking indexes with `amcheck` is not an alternative to having data checksums enabled. It is recommended to both
   enable data checksums and use `amcheck` corruption checks regularly (e.g., after provisioning of replicas, backup
   restore verification, or any time you suspect corruption in indexes).

5. If errors are found, it means there is corruption (unless `amcheck` has a bug). If errors are NOT found, it does NOT
   mean that there is no corruption – the corruption topic is very broad and no single tool can check for all possible
   kinds of corruption.

6. Potential cases when btree corruption might happen:

    - Postgres bugs (e.g. versions 14.0 – 14.3 had a bug in `REINDEX CONCURRENTLY`, potentially corrupting btree
      indexes)
    - Copying `PGDATA` from one OS version to another, silently switching to a different `glibc` version, introducing
      changes in
      collation rules. PG15+ produces a warning (`WARNING:  database XXX has a collation version mismatch`) in such
      cases,
      while older versions do not complain, so silent corruption risks are significant. It is always recommended to test
      upgrade and verify indexes with `amcheck`.
    - Performing a major upgrade, upgrading replicas using `rsync --data-only` without correctly handling the order in
      which the primary and replicas are stopped:
      [details](https://postgresql.org/message-id/flat/CAM527d8heqkjG5VrvjU3Xjsqxg41ufUyabD9QZccdAxnpbRH-Q%40mail.gmail.com).

7. Testing **GiST** and **GIN** for corruption is still in development, though there
   are [proposed patches](https://commitfest.postgresql.org/45/3733/) that can be used (with a note that they are not
   yet officially released).

8. Testing **unique indexes** for corruption is also in development.
