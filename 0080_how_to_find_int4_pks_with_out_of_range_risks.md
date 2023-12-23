Originally from: [tweet](https://twitter.com/samokhvalov/status/1735925615185002701), [LinkedIn post]().

---

# How to find int4 PKs with out-of-range risks in a large database

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Modern ORMs like Rails or Django use int8 (bigint) primary keys (PKs) today. However, in older projects, there may be
old tables with int4 (integer, int, serial) PKs, that have grown and have risks of int4 overflow – 
[max value for int4 is 2,147,483,647](https://postgresql.org/docs/current/datatype-numeric.html), 
and PK conversion int4->int8 without downtime is not a trivial task (TODO: cover it in another howto).

Here is how we can quickly check if there are tables with int2 or int4 PKs and how much of the "capacity" has been used
in each case (postgres-checkup query):

```sql
do $$
declare
  min_relpages int8 := 0; -- in very large DBs, skip small tables by setting this to 100 
  rec record;
  out text := '';
  out1 json;
  i numeric := 0;
  val int8;
  ratio numeric;
  sql text;
begin
  for rec in
    select
      c.oid,
      spcname as tblspace,
      nspname as schema_name,
      relname as table_name,
      t.typname,
      pg_get_serial_sequence(format('%I.%I', nspname, relname), attname) as seq,
      min(attname) as attname
    from pg_index i
    join pg_class c on c.oid = i.indrelid
    left join pg_tablespace tsp on tsp.oid = reltablespace
    left join pg_namespace n on n.oid = c.relnamespace
    join pg_attribute a on
      a.attrelid = i.indrelid
      and a.attnum = any(i.indkey)
    join pg_type t on t.oid = atttypid
    where
      i.indisprimary
      and (
        c.relpages >= min_relpages
        or pg_get_serial_sequence(format('%I.%I', nspname, relname), attname) is not null
      ) and t.typname in ('int2', 'int4')
      and nspname <> 'pg_toast'
      group by 1, 2, 3, 4, 5, 6
      having count(*) = 1 -- skip PKs with 2+ columns (TODO: analyze them too)
  loop
    raise debug 'table: %', rec.table_name;

    if rec.seq is null then
      sql := format('select max(%I) from %I.%I;', rec.attname, rec.schema_name, rec.table_name);
    else
      sql := format('select last_value from %s;', rec.seq);
    end if;

    raise debug 'sql: %', sql;
    execute sql into val;

    if rec.typname = 'int4' then
      ratio := (val::numeric / 2^31)::numeric;
    elsif rec.typname = 'int2' then
      ratio := (val::numeric / 2^15)::numeric;
    else
      assert false, 'unreachable point';
    end if;

    if ratio > 0.1 then -- report only if > 10% of capacity is reached
      i := i + 1;

      out1 := json_build_object(
        'table', (
          coalesce(nullif(quote_ident(rec.schema_name), 'public') || '.', '')
          || quote_ident(rec.table_name)
        ),
        'pk', rec.attname,
        'type', rec.typname,
        'current_max_value', val,
        'capacity_used_percent', round(100 * ratio, 2)
      );

      raise debug 'cur: %', out1;

      if out <> '' then
        out := out || e',\n';
      end if;

      out := out || format('  "%s": %s', rec.table_name, out1);
    end if;
  end loop;

  raise info e'{\n%\n}', out;
end;
$$ language plpgsql;
```

Output example:

```json
INFO:  {
  "oldbig": {"table" : "oldbig", "pk" : "id", "type" : "int4", "current_max_value" : 2107480000, "capacity_used_percent" : 98.14},
  "oldbig2": {"table" : "oldbig2", "pk" : "id", "type" : "int4", "current_max_value" : 1107480000, "capacity_used_percent" : 51.57}
}
```
