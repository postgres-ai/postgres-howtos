Originally from: [tweet](https://twitter.com/samokhvalov/status/1735568654996283501), [LinkedIn post]().

---

# How to estimate the YoY growth of a very large table using row creation timestamps and the planner statistics

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Assume we have a 10 TiB table created many years ago, partitioned or not, like this one:

```sql
create table t (
  id int8 primary key,   -- of course, not int4
  created_at timestamptz default now(),
  ...
);
```

and we need to quickly understand how the table grew year over year, assuming that no rows were deleted (or only a
negligible amount). So, we just need to count rows for every year.

A straightforward approach would be:

```sql
select
  date_trunc('year', created_at) as year,
  count(*)
from t
group by year
order by year;
```

However, for a 10 TiB table, we'd need to wait many hours, if not days, for this analysis to complete.

Here is fast, but not a precise way to get the row counts for each year (assuming the table has up-to-date stats; if in
doubt, run `ANALYZE` on it first):

```sql
do $$
declare
  table_fqn text := 'public.t';
  year_start int := 2000;
  year_end int := extract(year from now())::int;
  year int;
  explain_json json;
begin
  for year in year_start..year_end loop
    execute format(
      $e$
        explain (format json) select *
        from %s
        where created_at
        between '%s-01-01' and '%s-12-31'
      $e$,
      table_fqn,
      year,
      year
    ) into explain_json;

    raise info 'Year: %, Estimated rows: %',
      year,
      explain_json->0->'Plan'->>'Plan Rows';
  end loop;
end $$;
```
