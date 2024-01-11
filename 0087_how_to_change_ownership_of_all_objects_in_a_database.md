Originally from: [tweet](https://twitter.com/samokhvalov/status/1738464958386737401), [LinkedIn post]().

---

# How to change ownership of all objects in a database

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

If you need to change ownership of *all* database objects in current database, use this anonymous `DO`
block (or copy-paste from [here](https://gitlab.com/postgres-ai/database-lab/-/snippets/2075222)):

```sql
do $$
declare
  new_owner text := 'NEW_OWNER_ROLE_NAME';
  object_type record;
  r record;
  sql text;
begin
  -- New owner for all schemas
  for r in select * from pg_namespace loop
    sql := format(
      'alter schema %I owner to %I;',
      r.nspname,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;

  -- Various DB objects
  -- c: composite type
  -- p: partitioned table
  -- i: index
  -- r: table
  -- v: view
  -- m: materialized view
  -- S: sequence
  for object_type in
    select
      unnest('{type,table,table,view,materialized view,sequence}'::text[]) type_name,
      unnest('{c,p,r,v,m,S}'::text[]) code
  loop
    for r in
      execute format(
        $sql$
          select n.nspname, c.relname
          from pg_class c
          join pg_namespace n on
            n.oid = c.relnamespace
            and not n.nspname in ('pg_catalog', 'information_schema')
            and c.relkind = %L
          order by c.relname
        $sql$,
        object_type.code
      )
    loop
      sql := format(
        'alter %s %I.%I owner to %I;',
        object_type.type_name,
        r.nspname,
        r.relname,
        new_owner
      );

      raise debug 'Execute SQL: %', sql;

      execute sql;
    end loop;
  end loop;

  -- Functions, procedures
  for r in 
    select
      p.proname,
      n.nspname,
      pg_catalog.pg_get_function_identity_arguments(p.oid) as args
    from pg_catalog.pg_namespace as n
    join pg_catalog.pg_proc as p on p.pronamespace = n.oid
    where
      not n.nspname in ('pg_catalog', 'information_schema')
      and p.proname not ilike 'dblink%' -- We do not want dblink to be involved (exclusion)
  loop
    sql := format(
      'alter function %I.%I(%s) owner to %I', -- todo: check support CamelStyle r.args
      r.nspname,
      r.proname,
      r.args,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;

  -- Full text search dictionary
  -- TODO: text search configuration
  for r in 
    select * 
    from pg_catalog.pg_namespace n
    join pg_catalog.pg_ts_dict d on d.dictnamespace = n.oid
    where not n.nspname in ('pg_catalog', 'information_schema')
  loop
    sql := format(
      'alter text search dictionary %I.%I owner to %I',
      r.nspname,
      r.dictname,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;

  -- Domains
  for r in 
     select typname, nspname
     from pg_catalog.pg_type
     join pg_catalog.pg_namespace on pg_namespace.oid = pg_type.typnamespace
     where typtype = 'd' and not nspname in ('pg_catalog', 'information_schema')
  loop
    sql := format(
      'alter domain %I.%I owner to %I',
      r.nspname,
      r.typname,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;
end
$$;
```

Do not forget to change the value of `new_owner`.

The query excludes schemas `pg_catalog` and `information_schema`, and it covers: schema objects, tables, views,
materialized views, functions, text search dictionaries, and domains. Depending on your PG versions, there might be
other objects you need to address. Adjust the code if/as needed.

To see the debug messages, change `client_min_messages` before running:

```sql
set client_min_messages to debug;
```
