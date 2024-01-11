Originally from: [tweet](https://twitter.com/samokhvalov/status/1724721073508560940), [LinkedIn post]().

---

# Pre- and post-steps for benchmark iterations

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

When conducting a Postgres benchmark (see [Day 13: How to benchmark](0013_how_to_benchmark.md)), quite often we need to
run multiple benchmark iterations on the same setup. It may be reasonable to perform the same series of unified steps
before and after each iteration:

1. Before: flush the caches (or, conversely, warm them up).
2. Before: reset cumulative statistics.
3. After: save statistics and other forms of benchmark artefacts.

The approach described here allows conducting benchmarks in a unified way.

## Pre-step: flush the caches

As already mentioned, there are two possible preparation tactics:

- Warming up caches (either using [pg_prewarm](https://postgresql.org/docs/current/pgprewarm.html) or just performing
  the iteration's test steps multiple times and considering only the last one as a result). We won't cover it here.
- Flushing caches (the OS page cache and the Postgres buffer pool), so each iteration starts "on cold".

If the latter approach is chosen, it may also be reasonable to:

- increase the duration of each iteration to give the caches time to warm up, and
- review time series of metrics within each iteration (e.g., using `pgbench`'s option `-P 10` to see latency/throughput
  numbers every 10 seconds).

How to flush the caches:

- Postgres buffer pool – just restart Postgres:

    ```bash
    pg_ctl restart -D $PGDATA -m fast
    ```

- To free `pagecache`, `dentries` and `inodes`:

    ```bash
    sync # all buffered operations are written to disk
    echo 3 > /proc/sys/vm/drop_caches
    ```

## Pre-step: reset statistics

It may make sense to reset all available cumulative statistics in the cluster (depending on the current Postgres
version), including those that additional extensions such as the standard `pg_stat_statements` and
additional `pg_wait_samling`, `pg_stat_kcache`, `pg_qualstats`, if installed:

```sql
do $$
declare
  -- Main reset commands – a JSON object having the form:
  --                      {"command": min-PG-version-num}
  -- todo: in PG17+, there is pg_stat_reset_shared(null)
  reset_cmd_main json := $json${
    "pg_stat_reset()":                             90000,
    "pg_stat_reset_shared('bgwriter')":            90000,
    "pg_stat_reset_shared('archiver')":            90000,
    "pg_stat_reset_shared('io')":                 160000,
    "pg_stat_reset_shared('wal')":                140000,
    "pg_stat_reset_shared('recovery_prefetch')":  140000,
    "pg_stat_reset_slru(null)":                   130000
  }$json$;

  -- Extension commands - a JSON object having the form:
  --                      {"command": "extension name"}
  reset_cmd_et json := $json${
    "pg_stat_statements_reset()":         "pg_stat_statements",
    "pg_stat_kcache_reset()":             "pg_stat_kcache",
    "pg_wait_sampling_reset_profile()":   "pg_wait_sampling",
    "pg_qualstats_reset()":               "pg_qualstats"
  }$json$;

  cmd record;

  cur_ver int;
begin
  cur_ver := current_setting('server_version_num')::int;
  raise info 'Current PG version (num): %', cur_ver;

  -- Main reset commands
  for cmd in select * from json_each_text(reset_cmd_main) loop
    if cur_ver >= (cmd.value)::int then
      raise info 'Execute SQL: select %', cmd.key;

      execute format ('select %s', cmd.key);
    end if;
  end loop;

  -- Extension reset commands
  for cmd in select * from json_each_text(reset_cmd_et) loop
    if '' <> (
      select installed_version
      from pg_available_extensions
      where name = cmd.value
    ) then
      raise info 'Execute SQL: select %', cmd.key;

      execute format ('select %s', cmd.key);
    end if;
  end loop;
end
$$;
```

This step is worth doing right before the benchmark run, so statistics are clean and no preparation actions affect it.

## Post-step: gather artifacts

Multiple steps are recommended here.

1) Dump the content of all `pg_stat_*` views:

    ```bash
    for viewname in $(psql -tAXc "
        select relname
        from pg_catalog.pg_class
        where relkind = 'view' and relname like 'pg_stat%'" \
    ); do
      psql -Xc "copy (select * from ${viewname})
          to stdout with csv header delimiter ','" \
        > "${destination}/${viewname}.csv"
    done

    psql -Xc "copy (select * from pg_stat_kcache())
          to stdout with csv header delimiter ','" \
        > "${destination}/pg_stat_kcache.csv"

    psql -Xc "copy (
          select
            event_type as wait_type,
            event as wait_event,
            sum(count) as of_events
          from pg_wait_sampling_profile
          group by event_type, event
          order by of_events desc
        ) to stdout with csv header delimiter ','" \
      > "${destination}/pg_wait_sampling_profile.csv"

    # todo: pg_qualstats
    ```

2) Collect logs: Just compress and copy the files from the log directory.

3) Dump the actual configuration (`pg_settings`):

    ```bash
    psql -Xc "copy (
        select * from pg_settings order by name
      ) to stdout with csv header delimiter ','" \
      > "${destination}/pg_settings_all.csv"

    psql -Xc "
        select name, setting as current_setting, unit, boot_val as default, context
        from pg_settings
        where source <> 'default'" \
      > "${destination}/pg_settings_non_default.txt"
    ```

4) All other potentially useful artifacts, if needed: system log, sar data, etc.
