Originally from: [tweet](https://twitter.com/samokhvalov/status/1726184669622989250), [LinkedIn post]().

---

# How to check btree indexes for corruption (pg_amcheck)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

🥳🤩🎂🎉 It's my birthday, plus we've just soft-launched our
[postgres.ai bot](https://twitter.com/samokhvalov/status/1726177412755677283) – so forgive me a far from being complete article
this time. Nevertheless, I keep posting 😅

There are many types of corruption. Some kinds of them can be identified by extension `amcheck`
(see: [Using amcheck Effectively](https://postgresql.org/docs/current/amcheck.html#AMCHECK-USING-AMCHECK-EFFECTIVELY))
which is included to standard pack of "contrib modules" – so, just create it:

```sql
create extension amcheck;
```

For Postgres 14+, prefer using CLI tool [pg_amcheck](https://postgresql.org/docs/current/app-pgamcheck.html), one of
benefits of which is option `-j` (`--jobs=NUM`) – parallel workers to check multiple indexes faster.

An example of use (put your connection options and adjust the number of parallel workers):

```bash
pg_amcheck \
    {{ connection options }} \
    --install-missing \
    --jobs=8 \
    --verbose \
    --progress \
    --heapallindexed \
    --parent-check \
  2>&1 \
  | ts \
  | tee -a pg_amcheck.$(date "+%F-%H-%M").log
```

**IMPORTANT:** the options `--heapallindexed` and `--parent-check` trigger a long, but more advanced checking. The
`--parent-check` option is blocking writes (`UPDATE`s, etc.), so do not use it on production nodes that receive user
traffic. The option `--heapallindexed` increases the load and duration of the check, but can be used live with
care. Without both of these options the check performed will be light, potentially not noticing some issues
(read [Using amcheck Effectively](https://postgresql.org/docs/current/amcheck.html#AMCHECK-USING-AMCHECK-EFFECTIVELY)).

Once the snippet above is fully finished, check if the resulting log contains errors:

```bash
egrep 'ERROR:|DETAIL:|LOCATION:' \
 pg_amcheck.$(date "+%F-%H-%M").log
```

The indexes where corruption is detected need to be carefully analyzed and, if the problem is confirmed, re-created
(`REINDEX CONCURRENTLY`).
