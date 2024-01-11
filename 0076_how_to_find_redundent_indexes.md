Originally from: [tweet](https://twitter.com/samokhvalov/status/1734467240832201064), [LinkedIn post]().

---

# How to find redundant indexes

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Let's discuss today how to find and clean up redundant indexes.

## What is a redundant index?

Redundant indexes refer to multiple indexes on a table that serve the same purpose or where one index could satisfy the
queries that all others do. Two basic examples:

1. **Duplicate Indexes:** Two or more indexes that have exactly the same columns in the same order are clearly
   redundant.

2. **Overlapping Indexes:** When you have a multi-column index, for example on `(a, b)`, and then another index just on
   `(a)`. For queries that only filter on column `a`, the index on `(a, b)` could suffice, making the index on `(a)`
   potentially unnecessary.

Note that `(a)` is redundant to `(a, b)` but not to `(b, a)`. Accordingly, `(b)` is not redundant to `(a, b)`.

Further, we'll also assume that:

- unique indexes should not be considered in this kind of analysis because they have special purpose;
- indexes on expression follow the same rules – we just consider each expression similarly as columns, and expression
  values should be fully matched;
- in case of partial indexes, the conditions should be fully matched (this rule can be adjusted with certain
  assumptions, but we won't do that);
- we don't consider covering indexes (🎯 **TODO:** include them as well).

## Why to clean up redundant indexes

The same [6 reasons that we discussed for the unused indexes](0075_how_to_find_unused_indexes.md) are applicable here.

## General algorithm of unused indexes cleanup

1. Using the query provided below, identify the sets of redundant indexes. It is enough to analyze only one node in a
   cluster – e.g. the primary. It doesn't matter when stats were reset because this analysis is based only the static
   information (DB structure).

2. For each index that is considered redundant, perform a manual analysis to avoid mistakes. If not fully sure, remove
   such index from consideration.

3. Optionally, consider the "soft-drop" technique as described
   in [Day 53: Index maintenance](0053_index_maintenance.md), before actually dropping an index.

4. After proper analysis and testing, drop one index at a time, closely watching the database performance metrics to
   ensure there are no unwanted plan changes. To drop indexes, use `DROP INDEX CONCURRENTLY`.

## Query to find redundant indexes

Here is a query from
[postgres-checkup](https://gitlab.com/postgres-ai/postgres-checkup/-/blob/a6c8b6ae0772d5f7afb0a7004a740f6643a31aa2/resources/checks/H004_redundant_indexes.sh),
it produces a result in the form of JSON, including the commands to drop indexes using `DROP INDEX CONCURRENTLY`.

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
    and not i.indisunique
    and conkey is not null
    and ci.relpages > (select min_relpages from const)
    and si.idx_scan < 10
), index_data as ( -- Redundant indexes
  select
    *,
    indkey::text as columns,
    array_to_string(indclass, ', ') as opclasses
  from pg_index i
  join pg_class ci on ci.oid = i.indexrelid and ci.relkind = 'i'
  where indisvalid = true and ci.relpages > (select min_relpages from const)
), redundant_indexes as (
  select
    i2.indexrelid as index_id,
    tnsp.nspname AS schema_name,
    trel.relname AS table_name,
    pg_relation_size(trel.oid) as table_size_bytes,
    irel.relname AS index_name,
    am1.amname as access_method,
    (i1.indexrelid::regclass)::text as reason,
    i1.indexrelid as reason_index_id,
    pg_get_indexdef(i1.indexrelid) main_index_def,
    pg_size_pretty(pg_relation_size(i1.indexrelid)) main_index_size,
    pg_get_indexdef(i2.indexrelid) index_def,
    pg_relation_size(i2.indexrelid) index_size_bytes,
    s.idx_scan as index_usage,
    quote_ident(tnsp.nspname) as formated_schema_name,
    coalesce(nullif(quote_ident(tnsp.nspname), 'public') || '.', '') || quote_ident(irel.relname) as formated_index_name,
    quote_ident(trel.relname) AS formated_table_name,
    coalesce(nullif(quote_ident(tnsp.nspname), 'public') || '.', '') || quote_ident(trel.relname) as formated_relation_name,
    i2.opclasses
  from index_data as i1
  join index_data as i2 on
    i1.indrelid = i2.indrelid -- the same table
    and i1.indexrelid <> i2.indexrelid -- NOT the same index
  join pg_opclass op1 on i1.indclass[0] = op1.oid
  join pg_opclass op2 on i2.indclass[0] = op2.oid
  join pg_am am1 on op1.opcmethod = am1.oid
  join pg_am am2 on op2.opcmethod = am2.oid
  join pg_stat_user_indexes as s on s.indexrelid = i2.indexrelid
  join pg_class as trel on trel.oid = i2.indrelid
  join pg_namespace as tnsp on trel.relnamespace = tnsp.oid
  join pg_class as irel on irel.oid = i2.indexrelid
  where
    not i2.indisprimary -- index 1 is not PK
    and not ( -- skip if index1 is (primary or uniq) and is NOT (primary and uniq)
      i2.indisunique and not i1.indisprimary
    )
    and am1.amname = am2.amname -- same access type
    and i1.columns like (i2.columns || '%') -- index 2 includes all columns from index 1
    and i1.opclasses like (i2.opclasses || '%')
    -- index expressions is same
    and pg_get_expr(i1.indexprs, i1.indrelid) is not distinct from pg_get_expr(i2.indexprs, i2.indrelid)
    -- index predicates is same
    and pg_get_expr(i1.indpred, i1.indrelid) is not distinct from pg_get_expr(i2.indpred, i2.indrelid)
), redundant_indexes_fk as (
  select
    ri.*,
    (
      select count(1)
      from fk_indexes fi
      where
        fi.fk_table_ref = ri.table_name
        and fi.opclasses like (ri.opclasses || '%')
     ) > 0 as supports_fk
  from redundant_indexes ri
), redundant_indexes_tmp_num as ( -- Cut recursive links
  select row_number() over () num, rig.*
  from redundant_indexes_fk rig
), redundant_indexes_tmp_links as (
  select
    ri1.*,
    ri2.num as r_num
  from redundant_indexes_tmp_num ri1
  left join redundant_indexes_tmp_num ri2 on
    ri2.reason_index_id = ri1.index_id
    and ri1.reason_index_id = ri2.index_id
), redundant_indexes_tmp_cut as (
  select *
  from redundant_indexes_tmp_links
  where num < r_num or r_num is null
), redundant_indexes_cut_grouped as (
  select
    distinct(num),
    *
  from redundant_indexes_tmp_cut
  order by index_size_bytes desc
), redundant_indexes_grouped as (
  select
    index_id,
    schema_name,
    table_name,
    table_size_bytes,
    index_name,
    access_method,
    string_agg(distinct reason, ', ') as reason,
    string_agg(main_index_def, ', ') as main_index_def,
    string_agg(main_index_size, ', ') as main_index_size,
    index_def,
    index_size_bytes,
    index_usage,
    formated_index_name,
    formated_schema_name,
    formated_table_name,
    formated_relation_name,
    supports_fk
  from redundant_indexes_cut_grouped
  group by
    index_id,
    table_size_bytes,
    schema_name,
    table_name,
    index_name,
    access_method,
    index_def,
    index_size_bytes,
    index_usage,
    formated_index_name,
    formated_schema_name,
    formated_table_name,
    formated_relation_name,
    supports_fk
  order by index_size_bytes desc
), redundant_indexes_num as (
  select row_number() over () num, rig.*
  from redundant_indexes_grouped rig
), redundant_indexes_json as (
  select
    json_object_agg(coalesce(rin.schema_name, 'public') || '.' || rin.index_name, rin) as json
  from redundant_indexes_num rin
), redundant_indexes_total as (
  select
    sum(index_size_bytes) as index_size_bytes_sum,
    sum(table_size_bytes) as table_size_bytes_sum
  from redundant_indexes_grouped
), do_lines as (
  select
    format(
      'DROP INDEX CONCURRENTLY %s; -- %s, %s, table %s',
      formated_index_name,
      pg_size_pretty(index_size_bytes)::text,
      reason,
      formated_relation_name
    ) as line
  from redundant_indexes_grouped
  order by table_name, index_name
), undo_lines as (
  select
    replace(
      format('%s; -- table %s', index_def, formated_relation_name),
      'CREATE INDEX',
      'CREATE INDEX CONCURRENTLY'
    ) as line
  from redundant_indexes_grouped
  order by table_name, index_name
), database_stat as (
  select row_to_json(dbstat)
  from (
    select
      sd.stats_reset::timestamptz(0),
      age(
        date_trunc('minute',now()),
        date_trunc('minute',sd.stats_reset)
      ) as stats_age,
      ((extract(epoch from now()) - extract(epoch from sd.stats_reset))/86400)::int as days,
      (select pg_database_size(current_database())) as database_size_bytes
    from pg_stat_database sd
    where datname = current_database()
  ) as dbstat
) -- final result
select
  jsonb_pretty(jsonb_build_object(
    'redundant_indexes',
    (select * from redundant_indexes_json),
    'redundant_indexes_total',
    (select row_to_json(rit) from redundant_indexes_total as rit),
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
