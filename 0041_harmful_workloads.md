Originally from: [tweet](https://twitter.com/samokhvalov/status/1721397029979779140), [LinkedIn post]().

---

# How to break a database, Part 3: Harmful workloads

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

See also

- [Part 1: How to Corrupt](0039_how_to_break_a_database_part_1_how_to_corrupt.md).
- [Part 2: Simulate infamous transaction ID wraparound](0040_how_to_break_a_database_part_2_simulate_xid_wraparound.md).

## Too many connections

A simple snippet that creates 100 idle connections just with `psql` and a named pipe (a.k.a. FIFO, works in both macOS
and Linux):

```bash
mkfifo dummy

for i in $(seq 100); do
  psql -Xf dummy >/dev/null 2>&1 &
done

❯ psql -Xc 'select count(*) from pg_stat_activity'
  count
-------
  106
(1 row)
```

To close these connections, we can open a writing file descriptor to the FIFO and close it without writing any data:

```bash
  exec 3>dummy && exec 3>&-
```

Now the 100 extra connections have gone:

```bash
 ❯ psql -Xc 'select count(*) from pg_stat_activity'
  count
 -------
      6
 (1 row)
```

And if the number of connections reaches `max_connections` when we perform the steps above, we should see this when
trying to establish a new connection:

```bash
❯ psql
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  sorry, too many clients already
```

## Idle-in-transaction sessions

This recipe we used in the XID wraparound simulation:

```bash
mkfifo dummy

psql -Xc "
 set idle_in_transaction_session_timeout = 0;
 begin;
 select pg_current_xact_id()
 " \
-f dummy &
```

To release:

```bash
  exec 3>dummy && exec 3>&-
```

## More types of harm using various tools

This tool can help you simulate various harmful workloads:
[noisia – harmful workload generator for PostgreSQL](https://github.com/lesovsky/noisia).

As of 2023, it supports:

- idle transactions - active transactions on hot-write tables that do nothing during their lifetime
- rollbacks - fake invalid queries that generate errors and increase rollbacks counter
- waiting transactions - transactions that lock hot-write tables and then idle, leading to other transactions getting
  stuck
- deadlocks - simultaneous transactions where each holds locks that the other transactions want
- temporary files - queries that produce on-disk temporary files due to lack of `work_mem`
- terminate backends - terminate random backends (or queries) using `pg_terminate_backend()`, `pg_cancel_backend()`
- failed connections - exhaust all available connections (other clients unable to connect to Postgres)
- fork connections - execute single, short query in a dedicated connection (lead to excessive forking of Postgres
  backends)

And this tool will crash your database periodically: [pg_crash](https://github.com/cybertec-postgresql/pg_crash)

For Aurora users, there are interesting functions: `aurora_inject_crash()`, `aurora_inject_replica_failure()`,
`aurora_inject_disk_failure()`, `aurora_inject_disk_congestion()`: See
[Testing Amazon Aurora PostgreSQL by using fault injection queries](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Managing.FaultInjectionQueries.html).

## Summary

The whole topic of chaos engineering is interesting and, in my opinion, has a great potential in there are of
databases – to test recovery, failover, practice various incident situations. Some resources (beyond databases):

- Wikipedia: [Chaos engineering](https://en.wikipedia.org/wiki/Chaos_engineering)
- Netflix's [Chaos Monkey](https://github.com/Netflix/chaosmonkey), a resiliency tool that helps applications tolerate
  random instance failures

Ideally, mature processes of database administration. whether in the cloud or not, managed or not, should include:

1. Regular simulation of incidents in non-production, to practice and improve runbooks for incident mitigation.

2. Regular initiation of incidents in production to see how *actually* automated mitigation works. For example:
   auto-removal of a crashed replica, autofailover, alerts and team response to long-running transactions.
