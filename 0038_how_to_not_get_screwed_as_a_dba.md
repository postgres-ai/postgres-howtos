Originally from: [tweet](https://twitter.com/samokhvalov/status/1720360315899240720), [LinkedIn post]().

---

# How to NOT get screwed as a DBA (DBRE)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

The rules below are quite simple (but, often due to organizational reasons, not trivial to implement).

The following should be helpful for people responsible for databases in a fast-growing startup. Some items might be
useful for DBAs/DBREs in large companies.

## 1) Ensure that backup system is rock solid

1. Do not use `pg_dump` as a backup tool – instead, use a system with PITR (e.g., `pgBackRest`, `WAL-G`). If using a
   managed solution, learn in detail how backup systems are organized – do not trust blindly.

2. Learn about RPO and RTO, measure actual values and define target values. Cover values with monitoring (e.g., lagging
   `archive_command` should be considered as the highest priority incident).

3. TEST backups. This is the most critical part. An untested backup is a _Schrödinger's Backup_ – the condition of any
   backup is unknown until a restore is attempted. Automate testing.

4. Partial restore automation and speedup: in some cases, it is necessary to be able to recover a manually deleted data,
   not a whole database. In this case, to speed up recovery, consider options: (a) special delayed replica; (b) frequent
   cloud snapshots + PITR; (c) DBLab with hourly snapshots and PITR on a clone.

5. More topics to cover: proper encryption options, retention policy with old backups moved to a "colder" storage
   (cheaper), secondary location in a different cloud with very limited access.

Without doubt, backups are the most important topic in database administration. Getting screwed in this area is the
worst nightmare of any DBA. Pay maximum attention to backups, learn from other people's mistakes, not yours.

Reliable backup system is, perhaps, one of the biggest reasons why managed Postgres services are preferred in some
organizations. But again: don't trust blindly - study all the details, and test them yourself.

## 2) Corruption control

1. Enable [data checksums](0037_how_to_enable_data_checksums_without_downtime.md)

2. Be careful with OS / `glibc` upgrades – avoid index corruption.

3. Use `amcheck` to test indexes, heap, sequences.

4. Set up alerts to quickly react on error codes `XX000`, `XX001`, `XX002` in Postgres
   logs ([PostgreSQL Error Codes](https://postgresql.org/docs/current/errcodes-appendix.html))

5. There are many types of corruption – cover the low-hanging fruits and continue learning and implementing better
   control and corruption prevention measures (a set of good materials on the
   topic: [PostgreSQL Data Corruption and Bugs – Runbook](https://docs.google.com/spreadsheets/u/1/d/1zUH7IYOv46CVSmc-72CD7ROnMA6skJSQZjnm4yxvX9A/edit#gid=0))

## 3) HA

- Have standbys
- Consider using `synchronous_commit=remote_write` (or even `remote_apply`, depending on the case) and
  `synchronous_standby_names` ([Multiple Synchronous Standbys](https://postgresql.org/docs/current/warm-standby.html#SYNCHRONOUS-REPLICATION-MULTIPLE-STANDBYS))
- If self-managed, use **Patroni**. Otherwise, study all the HA options your provider offer and use them.

## 4) Performance

- Have a good monitoring
  system ([my PGCon slide deck on monitoring](https://twitter.com/samokhvalov/status/1664686535562625034))
- Set up advanced query analysis:
  `pg_stat_statements`, `pg_stat_kcache`, `pg_wait_sampling` / `pgsentinel`, `auto_explain`
- Build scalable "lab" environments and process for experimentation – DBA should not be a bottleneck for engineers to
  check ideas (spoiler: [@Database_Lab](https://twitter.com/Database_Lab) solves it).
- Implement capacity planning, make sure you have room for growth, perform proactive benchmarks.
- Architecture: microservices and sharding are all great and worth definitely considering, but the main question is:
  When? From the very beginning of the startup, or later? The answer is "it depends". Remember, with current Postgres
  performance and hardware any cloud offers, you can be fine and grow to dozens and hundreds of TiBs and hundreds of
  thousands of TPS. Choose your priorities – there are many successful stories with any approach.
- Don't hesitate to ask for help – community or paid consulting. Huge projects are running on Postgres, and there is a
  lot of experience accumulated.

## 5) Learn from other people's mistakes

- Postgres still has 32-bit transaction IDs. Make sure you don't hit the transaction ID (and multi-XID) wraparound –
  [Sentry's case](https://blog.sentry.io/transaction-id-wraparound-in-postgres/) and
  [Mailchimp's one](https://mailchimp.com/what-we-learned-from-the-recent-mandrill-outage/) are good lessons.
- Subtransactions – I personally consider their use dangerous in heavily-loaded systems and recommend
  [avoiding them](0035_how_to_use_subtransactions_in_postgres.md).
- Tune `autovacuum` – don't allow Postgres to accumulate a lot of dead tuples and/or bloat (yes, these two are different
  things). Turning off `autovacuum` is a good way to put your server down.
- In the OLTP context, avoid long-running transactions and unused/lagging replication slots.
- Learn how to deploy schema changes without downtime.
  Useful articles:
   - [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)
   - [Common DB schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes)
