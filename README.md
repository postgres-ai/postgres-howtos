This project has been started by @NikolayS on 2023-09-26 https://twitter.com/samokhvalov/status/1706748070967624174:

> I'm going to start a PostgreSQL marathon: each day I'll be posting a new "howto" recipe. Today is the day zero, and the first post is here.

> My goal is to create at least 365 posts 😎

> Why am I doing it?
> 1. Postgres docs are awesome but often lack practical pieces of advice (howtos)
> 2. 20+ years of database experience, from small startups to giants like Chewy, GitLab, Miro - always have a feeling that I need to share
> 3. eventually I aim to have a structured set of howtos, constantly improving it - and make the systems we develop at [Postgres.ai](https://Postgres.ai) / [Database_Lab](https://twitter.com/Database_Lab) better and more helpful.

[Subscribe](https://twitter.com/samokhvalov/status/1706748070967624174), like, share, and wish me luck with this -- and let's go! 🏊

## ToC
<!-- To build it using LLM, use this prompt:
"""
I want to build ToC for these files in Git. help.
Notice the numbers in the beginning 0001, 0002, etc.
Don't provide any additional comments – I need just the data to insert to README.
Format the response as markdown.

{{ and here provide the output of:  `grep "^ *# " *.md` }}

As an example, first 2 rows:
- 0001 [`EXPLAIN ANALYZE` or `EXPLAIN (ANALYZE, BUFFERS)`?](./0001_explain_analyze_buffers.md)
- 0002 [How to troubleshoot and speed up Postgres stop and restart attempts](./0002_how_to_troubleshoot_and_speedup_postgres_restarts.md)
"""
-->

- 0001 [`EXPLAIN ANALYZE` or `EXPLAIN (ANALYZE, BUFFERS)`?](./0001_explain_analyze_buffers.md)
- 0002 [How to troubleshoot and speed up Postgres stop and restart attempts](./0002_how_to_troubleshoot_and_speedup_postgres_restarts.md)
- 0003 [How to troubleshoot long Postgres startup](./0003_how_to_troubleshoot_long_startup.md)
- 0004 [Understanding how sparsely tuples are stored in a table](./0004_tuple_sparsenes.md)
- 0005 [How to work with pg_stat_statments, part 1](./0005_pg_stat_statements_part_1.md)
- 0006 [How to work with pg_stat_statements, part 2](./0006_pg_stat_statements_part_2.md)
- 0007 [How to work with pg_stat_statements, part 3](./0007_pg_stat_statements_part_3.md)
- 0008 [How to speed up pg_dump when dumping large databases](./0008_how_to_speed_up_pg_dump.md)
- 0009 [How to understand LSN values and WAL filenames](./0009_lsn_values_and_wal_filenames.md)
- 0010 [How to troubleshoot Postgres performance using FlameGraphs and eBPF (or perf)](./0010_flamegraphs_for_postgres.md)
- 0011 [Ad-hoc monitoring](./0011_ad_hoc_monitoring.md)
- 0012 [How to find query examples for problematic pg_stat_statements records](./0012_from_pgss_to_explain__how_to_find_query_examples.md)
- 0013 [How to benchmark](./0013_how_to_benchmark.md)
- 0014 [How to decide when query is too slow and needs optimization](./0014_how_to_decide_if_query_too_slow.md)
- 0015 [How to monitor CREATE INDEX / REINDEX progress in Postgres 12+](./0015_how_to_monitor_index_operations.md)
- 0016 [How to get into trouble using some Postgres features](./0016_how_to_get_into_trouble_using_some_postgres_features.md)
- 0017 [How to determine the replication lag](./0017_how_to_determine_the_replication_lag.md)
- 0018 [Over-indexing](./0018_over_indexing.md)
- 0019 [How to import CSV to Postgres](./0019_how_to_import_csv_to_postgres.md)
- 0020 [How to use pg_restore](./0020_how_to_use_pg_restore.md)
- 0021 [How to set application_name without extra queries](./0021_how_to_set_application_name_without_extra_queries.md)
- 0022 [How to analyze heavyweight locks, part 1](./0022_how_to_analyze_heavyweight_locks_part_1.md)
- 0023 [How to use OpenAI APIs right from Postgres to implement semantic search and GPT chat](./0023_how_to_use_openai_apis_in_postgres.md)
- 0024 [How to work with metadata](./0024_how_to_work_with_metadata.md)
- 0025 [How to quit from psql](./0025_how_to_quit_from_psql.md)
- 0026 [How to check btree indexes for corruption](./0026_how_to_check_btree_indexes_for_corruption.md)
- 0027 [How to compile Postgres on Ubuntu 22.04](./0027_how_to_compile_postgres_on_ubuntu_22.04.md)
- 0028 [How to work with arrays, part 1](./0028_how_to_work_with_arrays_part_1.md)
- 0029 [How to work with arrays, part 2](./0029_how_to_work_with_arrays_part_2.md)
- 0030 [How to deal with long-running transactions (OLTP)](./0030_how_to_deal_with_long-running_transactions_oltp.md)
- 0031 [How to troubleshoot a growing pg_wal directory](./0031_how_to_troubleshoot_a_growing_pg_wal_directory.md)
- 0032 [How to speed up bulk load](./0032_how_to_speed_up_bulk_load.md)
- 0033 [How to redefine a PK without downtime](./0033_how_to_redefine_a_PK_without_downtime.md)
- 0034 [How to perform initial / rough Postgres tuning](./0034_how_to_perform_postgres_tuning.md)
- 0035 [How to use subtransactions in Postgres](./0035_how_to_use_subtransactions_in_postgres.md)
- 0036 ["Find-or-insert" using a single query](./0036_find-or-insert_using_a_single_query.md)
- 0037 [How to enable data checksums without downtime](./0037_how_to_enable_data_checksums_without_downtime.md)
- 0038 [How to NOT get screwed as a DBA (DBRE)](./0038_how_to_not_get_screwed_as_a_dba.md)
- 0039 [How to break a database, Part 1: How to corrupt](./0039_how_to_break_a_database_part_1_how_to_corrupt.md)
- 0040 [How to break a database, Part 2: Simulate infamous transaction ID wraparound](./0040_how_to_break_a_database_part_2_simulate_xid_wraparound.md)
- 0041 [How to break a database, Part 3: Harmful workloads](./0041_harmful_workloads.md)
- 0042 [How to analyze heavyweight locks, part 2: Lock trees (a.k.a. "lock queues", "wait queues", "blocking chains")](./0042_how_to_analyze_heavyweight_locks_part_2.md)
- 0043 [How to format SQL](./0043_how_to_format_sql.md)
- 0044 [How to monitor transaction ID wraparound risks](./0044_how_to_monitor_transaction_id_wraparound_risks.md)
- 0045 [How to monitor xmin horizon to prevent XID/MultiXID wraparound and high bloat](./0045_how_to_monitor_xmin_horizon.md)
- 0046 [How to deal with bloat](./0046_how_to_deal_with_bloat.md)
- 0047 [How to install Postgres 16 with plpython3u: Recipes for macOS, Ubuntu, Debian, CentOS, Docker](./0047_how_to_install_postgres_16_with_plpython3u.md)
- 0048 [How to generate fake data](./0048_how_to_generate_fake_data.md)
- 0049 [How to use variables in psql scripts](./0049_how_to_use_variables_in_psql_scripts.md)
- 0050 [Pre- and post-steps for benchmark iterations](./0050_pre_and_post_steps_for_benchmark_iterations.md)
- 0051 [Learn how to work with schema metadata by spying after psql](./0051_learn_about_schema_metadata_via_psql.md)
- 0052 [How to reduce WAL generation rates](./0052_how_to_reduce_wal_generation_rates.md)
- 0053 [Index maintenance](./0053_index_maintenance.md)
- 0054 [How to check btree indexes for corruption (pg_amcheck)](./0054_how_to_check_btree_indexes_for_corruption.md)
- 0055 [How to drop a column](./0055_how_to_drop_a_column.md)
- 0056 [How to make the non-production Postgres planner behave like in production](./0056_how_to_imitate_production_planner.md)
- 0057 [How to convert a physical replica to logical](./0057_how_to_convert_a_physical_replica_to_logical.md)
- 0058 [How to use Docker to run Postgres](./0058_how_to_use_docker_to_run_postgres.md)
- 0059 [psql tuning](./0059_psql_tuning.md)
- 0060 [How to add a column](./0060_how_to_add_a_column.md)
- 0061 [How to create an index, part 1](./0061_how_to_create_an_index_part_1.md)
- 0062 [How to create an index, part 2](./0062_how_to_create_an_index_part_2.md)
- 0063 [How to help others](./0063_how_to_help_others.md)
- 0064 [How to use UUID](./0064_how_to_use_uuid.md)
- 0065 [UUID v7 and partitioning (TimescaleDB)](./0065_uuid_v7_and_partitioning_timescaledb.md)
- 0066 [How many tuples can be inserted in a page](./0066_how_many_tuples_can_be_inserted_in_a_page.md)
- 0067 [Autovacuum "queue" and progress](./0067_autovacuum_queue_and_progress.md)
- 0068 [psql shortcuts](./0068_psql_shortcuts.md)
- 0069 [How to add a CHECK constraint without downtime](./0069_howd_tod_addd_ad_checkd_constraintd_withoutd_downtime.md)
- ...

## Contributors

- Some tweets converted to markdown by @msdousti (thanks!)
