Originally from: [tweet](https://twitter.com/samokhvalov/status/1734044926755967163), [LinkedIn post]().

---

# How to find unused indexes

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## Why to clean up unused indexes

Having unused indexes is bad because:

1. They occupy extra space on disk.

2. They slow down many write operations (`INSERT`s and non-HOT `UPDATE`s).

3. They "spam" caches (OS page cache, Postgres buffer pool) because index blocks are loaded there during write
   operations mentioned above.

4. Due to the same reasons, they "spam" WAL (thus, replication and backup systems need to handle more data).

5. Every index slows down the query planning
   (see
   [benchmarks](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/41#note_1602598974)).

6. Finally, if there is no plan caching (prepared statements are not used), each extra index increases chances to
   reach `FP_LOCK_SLOTS_PER_BACKEND=16` relations (both tables and indexes) involved in query processing, which, under
   high QPS, increases chances of having `LWLock:LockManager` contention (see
   [benchmarks](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/41)).

Thus, having a routine procedure for periodic analysis and cleanup of unused indexes is highly recommended.

## General algorithm of unused indexes cleanup

1. Using one of the queries provided below (all of them deal with `pg_stat_user_indexes`, where index usage stats can be
   found), identify unused indexes.

2. Make sure that the stats are old enough, inspecting timestamp `stats_reset` in `pg_stat_database` for your database.
   For example, they represent usage data for at least 1 month (depending on the application, this threshold can be more
   or less). If this condition is not met, postpone the analysis.

3. If replicas receive read-only traffic, analyze all of them as well, paying attention to the stats reset timestamps
   too.

4. If you have more than one production instance of the database (e.g., you develop a system that is installed in
   various places and all of them have their own databases), analyze as many database instances as possible, using steps
   1-3 above.

5. As a result, build a list of indexes that can be reliably named as "unused" – we know that we didn't use them during
   significant time, neither on the primary nor replicas, on all production systems we can observe.

6. For each index in the list drop it using `DROP INDEX CONCURRENTLY`.

## Queries to analyze unused indexes

A basic query is simple:

```sql
select *
from pg_stat_user_indexes
where idx_scan = 0;
```

How to understand how long ago the stats were last reset:

```sql
select
  stats_reset,
  age(now(), stats_reset)
from pg_stat_database
where datname = current_database();
```

For holistic analysis, you can use this query from
[postgres-checkup](https://gitlab.com/postgres-ai/postgres-checkup/-/blob/7f5c45b18b04c665d11f744486db047a1dbc680e/resources/checks/H002_unused_indexes.sh),
which also includes the analysis of rarely used indexes, DDL commands for index cleanup, and presents the result as JSON
for easier integration in observability and automation systems:

```sql
with const as (
  select 0 as min_relpages -- on very large DBs, increase it to, say, 100
), fk_indexes as (
  select
    n.nspname as schema_name,
    ci.relname as index_name,
    cr.relname as table_name,
    (confrelid::regclass)::text as fk_table_ref,
    array_to_string(indclass, ', ') as opclasses
  from pg_index i
  join pg_class ci on ci.oid = i.indexrelid and ci.relkind = 'i'
  join pg_class cr on cr.oid = i.indrelid and cr.relkind = 'r'
  join pg_namespace n on n.oid = ci.relnamespace
  join pg_constraint cn on cn.conrelid = cr.oid
  left join pg_stat_user_indexes si on si.indexrelid = i.indexrelid
  where
     contype = 'f'
     and i.indisunique is false
     and conkey is not null
     and ci.relpages > (select min_relpages from const)
     and si.idx_scan < 10
), table_scans as (
  select
    relid,
    tables.idx_scan + tables.seq_scan as all_scans,
    (tables.n_tup_ins + tables.n_tup_upd + tables.n_tup_del) as writes,
    pg_relation_size(relid) as table_size
  from pg_stat_user_tables as tables
  join pg_class c on c.oid = relid
  where c.relpages > (select min_relpages from const)
), all_writes as (
  select sum(writes) as total_writes
  from table_scans
), indexes as (
  select
    i.indrelid,
    i.indexrelid,
    n.nspname as schema_name,
    cr.relname as table_name,
    ci.relname as index_name,
    quote_ident(n.nspname) as formated_schema_name,
    coalesce(nullif(quote_ident(n.nspname), 'public') || '.', '') || quote_ident(ci.relname) as formated_index_name,
    quote_ident(cr.relname) as formated_table_name,
    coalesce(nullif(quote_ident(n.nspname), 'public') || '.', '') || quote_ident(cr.relname) as formated_relation_name,
    si.idx_scan,
    pg_relation_size(i.indexrelid) as index_bytes,
    ci.relpages,
    (case when a.amname = 'btree' then true else false end) as idx_is_btree,
    pg_get_indexdef(i.indexrelid) as index_def,
    array_to_string(i.indclass, ', ') as opclasses
  from pg_index i
  join pg_class ci on ci.oid = i.indexrelid and ci.relkind = 'i'
  join pg_class cr on cr.oid = i.indrelid and cr.relkind = 'r'
  join pg_namespace n on n.oid = ci.relnamespace
  join pg_am a ON ci.relam = a.oid
  left join pg_stat_user_indexes si on si.indexrelid = i.indexrelid
  where
    i.indisunique = false
    and i.indisvalid = true
    and ci.relpages > (select min_relpages from const)
), index_ratios as (
  select
    i.indexrelid as index_id,
    i.schema_name,
    i.table_name,
    i.index_name,
    idx_scan,
    all_scans,
    round(
      case
        when all_scans = 0 then 0.0::numeric
        else idx_scan::numeric / all_scans * 100
      end,
      2
    ) as index_scan_pct,
    writes,
    round(
      case
        when writes = 0 then idx_scan::numeric
        else idx_scan::numeric / writes
      end,
      2
    ) as scans_per_write,
    index_bytes as index_size_bytes,
    table_size as table_size_bytes,
    i.relpages,
    idx_is_btree,
    index_def,
    formated_index_name,
    formated_schema_name,
    formated_table_name,
    formated_relation_name,
    i.opclasses,
    (
      select count(1)
      from fk_indexes fi
      where
        fi.fk_table_ref = i.table_name
        and fi.schema_name = i.schema_name
        and fi.opclasses like (i.opclasses || '%')
    ) > 0 as supports_fk
  from indexes i
  join table_scans ts on ts.relid = i.indrelid
), never_used_indexes as ( -- Never used indexes
  select
    'Never Used Indexes' as reason,
    ir.*
  from index_ratios ir
  where
    idx_scan = 0
    and idx_is_btree
  order by index_size_bytes desc
), never_used_indexes_num as (
  select
    row_number() over () num,
    nui.*
  from never_used_indexes nui
), never_used_indexes_total as (
  select
    sum(index_size_bytes) as index_size_bytes_sum,
    sum(table_size_bytes) as table_size_bytes_sum
  from never_used_indexes
), never_used_indexes_json as (
  select
    json_object_agg(coalesce(nuin.schema_name, 'public') || '.' || nuin.index_name, nuin) as json
  from never_used_indexes_num nuin
), rarely_used_indexes as ( -- Rarely used indexes
  select
    'Low Scans, High Writes' as reason,
    *,
    1 as grp
  from index_ratios
  where
    scans_per_write <= 1
    and index_scan_pct < 10
    and idx_scan > 0
    and writes > 100
    and idx_is_btree
  union all
  select
    'Seldom Used Large Indexes' as reason,
    *,
    2 as grp
  from index_ratios
  where
    index_scan_pct < 5
    and scans_per_write > 1
    and idx_scan > 0
    and idx_is_btree
    and index_size_bytes > 100000000
  union all
  select
    'High-Write Large Non-Btree' as reason,
    index_ratios.*,
    3 as grp
  from index_ratios, all_writes
  where
    (writes::numeric / ( total_writes + 1 )) > 0.02
    and not idx_is_btree
    and index_size_bytes > 100000000
  order by grp, index_size_bytes desc
), rarely_used_indexes_num as (
  select row_number() over () num, rui.*
  from rarely_used_indexes rui
), rarely_used_indexes_total as (
  select
    sum(index_size_bytes) as index_size_bytes_sum,
    sum(table_size_bytes) as table_size_bytes_sum
  from rarely_used_indexes
), rarely_used_indexes_json as (
  select
    json_object_agg(coalesce(ruin.schema_name, 'public') || '.' || ruin.index_name, ruin) as json
  from rarely_used_indexes_num ruin
), do_lines as (
  select 
    format(
      'DROP INDEX CONCURRENTLY %s; -- %s, %s, table %s',
      formated_index_name,
      pg_size_pretty(index_size_bytes)::text,
      reason,
      formated_relation_name
    ) as line
  from never_used_indexes nui
  order by table_name, index_name
), undo_lines as (
  select
    replace(
      format('%s; -- table %s', index_def, formated_relation_name),
      'CREATE INDEX',
      'CREATE INDEX CONCURRENTLY'
    ) as line
  from never_used_indexes nui
  order by table_name, index_name
), database_stat as (
  select
    row_to_json(dbstat)
  from (
    select
      sd.stats_reset::timestamptz(0),
      age(
        date_trunc('minute', now()),
        date_trunc('minute', sd.stats_reset)
      ) as stats_age,
      ((extract(epoch from now()) - extract(epoch from sd.stats_reset)) / 86400)::int as days,
      (select pg_database_size(current_database())) as database_size_bytes
    from pg_stat_database sd
    where datname = current_database()
  ) dbstat
)
select
  jsonb_pretty(jsonb_build_object(
    'never_used_indexes',
    (select * from never_used_indexes_json),
    'never_used_indexes_total',
    (select row_to_json(nuit) from never_used_indexes_total as nuit),
    'rarely_used_indexes',
    (select * from rarely_used_indexes_json),
    'rarely_used_indexes_total',
    (select row_to_json(ruit) from rarely_used_indexes_total as ruit),
    'do',
    (select json_agg(dl.line) from do_lines as dl),
    'undo',
    (select json_agg(ul.line) from undo_lines as ul),    
    'database_stat',
    (select * from database_stat),
    'min_index_size_bytes',
    (select min_relpages * current_setting('block_size')::numeric from const)
  ));
```
