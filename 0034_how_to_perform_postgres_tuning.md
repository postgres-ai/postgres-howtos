Originally from: [tweet](https://twitter.com/samokhvalov/status/1718886593153630663), [LinkedIn post]().

---

# How to perform initial / rough Postgres tuning

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Modern Postgres provides more than 300 settings (a.k.a. GUC variables – "grand unified configuration"). Fine-tuning
Postgres for a particular environment, database, and workload is a very complex task.

But in most cases, the Pareto principle (a.k.a. rule 80/20) works pretty well: you need to spend limited effort to
address basic areas of tuning, and then focus on query performance. The reasoning behind this approach is simple and
solid: yes, you can spend a lot of effort and find a better value of `shared_buffers` than the traditional 25% (which,
as many people think, is far from ideal: e.g.,
see [Andres Freund's Tweet](https://twitter.com/andresfreundtec/status/1178765225895399424)), and then
find yourself in a position where a few queries with suboptimal performance – e.g., lacking proper indexes – ruin all
the positive effect from that fine-tuning.

Therefore, I recommend this approach:

1. Basic "rough" tuning
2. Log-related settings
3. Autovacuum tuning
4. Checkpointer tuning
5. Then focus on query optimization, reactive or proactive, and fine-tuning for specific areas only when there is a
   strong reason for it

## Basic rough tuning

For initial rough tuning, the empirical tools are "good enough" in most cases, following the 80/20 principle (actually,
perhaps even 95/5 in this case):

- [PGTune](https://pgtune.leopard.in.ua) ([source code](https://github.com/le0pard/pgtune))
- [PostgreSQL Configurator](https://pgconfigurator.cybertec.at)
- for TimescaleDB users: [timescaledb-tune](https://github.com/timescale/timescaledb-tune)

Additionally, to the official docs, [this resource](https://postgresqlco.nf) is good to use as a reference (it has
integrated information from various sources, not only official docs) – for example, check the page for
[random_page_cost](https://postgresqlco.nf/doc/en/param/random_page_cost/), a parameter which is quite often forgotten.

If you use a managed Postgres service such as RDS, quite likely this level of tuning is performed already when you
provision a server. But it's still worth double-checking – for example, some providers provision a server with SSD disk
but leave `random_page_cost` default – `4` – which is an outdated value suitable for magnetic disks. Just set it to `1`
if you have an SSD.

It is important to perform this level of tuning before query optimization efforts, because otherwise, you might need to
re-do query optimization once you adjusted the basic configuration.

## Log-related settings

A general rule here: the more logging, the better. Of course, assuming that you avoid saturation of two kinds:

- disk space (logs filled the disk)
- disk IO (too many writes per second caused by logging)

In short my recommendations are (this is worth a separate detailed post):

- turn on checkpoint logging, `log_checkpoints='on'`  (fortunately, it's on by default in PG15+),
- turn on all autovacuum logging, `log_autovacuum_min_duration=0` (or a very low value)
- log temporary files except tiny ones (e.g., `log_temp_files = 100`)
- log all DDL statements `log_statement='ddl'`
- adjust `log_line_prefix`
- set a low value for `log_min_duration_statement` (e.g., `500ms`) and/or use `auto_explain` to log slow queries with
  plans

## Autovacuum tuning

This is a big topic worth a separate post. In short, the key idea is that default settings don't suit for any modern
OLTP case (web/mobile apps), so autovacuum has to be always tuned. If we don't do it, autovacuum becomes a "converter"
of large portions of dead tuples to bloat, and this eventually negatively affects performance.

Two areas of tuning needs to be addressed:

1. Increase the frequency of processing – lowering `**_scale_factor` / `**_threshold` settings, we make autovacuum
   workers process tables when quite low value of dead tuples is accumulated
2. Allocate more resources for processing: more autovacuum workers (`autovacuum_workers`), more memory
   (`autovacuum_work_mem`), and higher "quotas" for work (controlled via `**_cost_limit` / `**_cost_delay`).

## Checkpointer tuning

Again, it's worth a separate post. But in short, you need to consider raising `checkpoint_timeout` and – most
importantly – `max_wal_size` (whose default is very small for modern machines and data volumes, just `1GB`), so
checkpoints occur less frequently, especially when a lot of writes happen. However, shifting settings in this direction
mean longer recovery time in case of a crash or recovery from backups – this is a trade-off that needs to be analyzed
for a particular case.

That's it. Generally, this initial/rough tuning of Postgres config shouldn't take long. For a particular cluster of type
of clusters, it's a 1-2 day work for an engineer. You don't actually need AI for this, empirical tools work well –
unless you do aim to squeeze 5-10% more (you might want it though, e.g., if you have thousands of servers).
