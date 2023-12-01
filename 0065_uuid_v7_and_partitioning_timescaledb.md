Originally from: [tweet](https://twitter.com/samokhvalov/status/1730107298369171943), [LinkedIn post]().

---

# UUID v7 and partitioning (TimescaleDB)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Ok, you've asked – here it is, a draft recipe to use UUIDv7 and partitioning (we'll
use [@TimescaleDB](https://twitter.com/TimescaleDB)). It's not super elegant, might be not the best, and requires some
effort to reach efficient plans (with partition pruning involved). If you have an alternative or improvement ideas in
mind – let me know.

We'll take the function by [@DanielVerite](https://twitter.com/DanielVerite) to generate UUIDv7 as basis:

```sql
create or replace function uuid_generate_v7() returns uuid
as $$
  -- use random v4 uuid as starting point (which has the same variant we need)
  -- then overlay timestamp
  -- then set version 7 by flipping the 2 and 1 bit in the version 4 string
select encode(
  set_bit(
    set_bit(
      overlay(
        uuid_send(gen_random_uuid())
        placing substring(int8send(floor(extract(epoch from clock_timestamp()) * 1000)::bigint) from 3)
        from 1 for 6
      ),
      52, 1
    ),
    53, 1
  ),
  'hex')::uuid;
$$ language SQL volatile;
```

## Helper functions, UUIDv7 <-> timestamptz

Next, we'll create two functions:

1. `ts_to_uuid_v7` – generate UUIDv7 based on any arbitrary `timestamptz` value, and
2. `uuid_v7_to_ts` – extract `timestamptz` from the existing UUIDv7 value.

Note that this approach is not what the authors of revised RFC4122 (that will likely be finalized soon) would encourage;
see [this discussion and the words](https://postgresql.org/message-id/flat/C80B8FDB-8D9E-48A2-82A2-48863987A1B1%40yandex-team.ru#074a05d31c9ce38bee2f8c8097877485)
by
[@x4mmmmmm](https://twitter.com/x4mmmmmm):

> ... as far as I know, RFC discourages extracting timestamps from UUIDs.

Anyway, let's just do it:

```sql
create extension pgcrypto;

create or replace function ts_to_uuid_v7(timestamptz) returns uuid
as $$
  select encode(
    set_bit(
      set_bit(
        overlay(
          uuid_send(gen_random_uuid())
          placing substring(int8send(floor(extract(epoch from $1) * 1000)::bigint) from 3)
          from 1 for 6
        ),
        52, 1
      ),
      53, 1
    ),
    'hex')::uuid;
$$ language SQL volatile;

create or replace function uuid_v7_to_ts(uuid_v7 uuid) returns timestamptz
as $$
  select
    to_timestamp(
      (
        'x' || substring(
          encode(uuid_send(uuid_v7), 'hex')
          from 1 for 12
        )
      )::bit(48)::bigint / 1000.0
    )::timestamptz;
$$ language sql;
```

Checking the functions:

```sql
test=# select now(), ts_to_uuid_v7(now() - interval '1y');
              now              |            ts_to_uuid_v7
-------------------------------+--------------------------------------
 2023-11-30 05:36:32.205093+00 | 0184c709-63cd-7bd1-99c3-a4773ab1e697
(1 row)

test=# select uuid_v7_to_ts('0184c709-63cd-7bd1-99c3-a4773ab1e697');
       uuid_v7_to_ts
----------------------------
 2022-11-30 05:36:32.205+00
(1 row)
```

Pretending that we haven't noticed the loss of microseconds, we continue.

> 🎯 **TODO:** : 
> 1) may it be the case when we need that precision? 
> 2) timezones

## Hypertable

Create a table, where we'll store ID as UUID, but additionally have a `timestamptz` column – this column will be used as
partitioning key when we convert the table to partitioned table ("hypertable" in TimescaleDB's terminology):

```sql
create table my_table (
  id uuid not null
    default '00000000-0000-0000-0000-000000000000'::uuid,
  payload text,
  uuid_ts timestamptz not null default clock_timestamp() -- or now(), depending on goals
);
```

The default value `00000000-...00` for "id" is "fake" – it will always be replaced in trigger, based on the timestamp:

```sql
create or replace function t_update_uuid() returns trigger
as $$
begin
  if new.id is null or new.id = '00000000-0000-0000-0000-000000000000'::uuid then
    new.id := ts_to_uuid_v7(new.uuid_ts);
  end if;

  return new;
end;
$$ language plpgsql;

create trigger t_update_uuid_before_insert_update
before insert or update on my_table
for each row execute function t_update_uuid();
```

Now, use TimescaleDB partitioning:

```sql
create extension timescaledb;

select create_hypertable(
  relation := 'my_table',
  time_column_name := 'uuid_ts',
  -- !! very small interval is just for testing
  chunk_time_interval := '1 minute'::interval
);
```

## Test data - fill the chunks

And now insert some test data – some rows for the "past" and some "current" rows:

```sql
insert into my_table(payload, uuid_ts)
select random()::text, ts
from generate_series(
  timestamptz '2000-01-01 00:01:00',
  timestamptz '2000-01-01 00:05:00',
  interval '5 second' 
) as ts;

insert into my_table(payload)
select random()::text
from generate_series(1, 10000);

vacuum analyze my_table;
```

Checking the structure of `my_table` in psql using `\d+` we now see that multiple partitions ("chunks") were created by
TimescaleDB:

```sql
test=# \d+ my_table
...
Child tables: _timescaledb_internal._hyper_2_3_chunk,
              _timescaledb_internal._hyper_2_4_chunk,
              _timescaledb_internal._hyper_2_5_chunk,
              _timescaledb_internal._hyper_2_6_chunk,
              _timescaledb_internal._hyper_2_7_chunk,
              _timescaledb_internal._hyper_2_8_chunk,
              _timescaledb_internal._hyper_2_9_chunk
```

## Test queries – partition pruning

Now we just need to remember that `created_at` should always participate in queries, to let planner deal with as few
partitions as possible – but knowing the `id` values, we can always reconstruct the `created_at` values, using 
`uuid_v7_to_ts()`:

```sql
test=# explain select * from my_table where created_at = uuid_v7_to_ts('00dc6ad0-9660-7b92-a95e-1d7afdaae659');
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Append  (cost=0.14..8.16 rows=1 width=41)
   ->  Index Scan using _hyper_5_11_chunk_my_table_created_at_idx on _hyper_5_11_chunk  (cost=0.14..8.15 rows=1 width=41)
         Index Cond: (created_at = '2000-01-01 00:01:00+00'::timestamp with time zone)
(3 rows)

test=# explain select * from my_table
  where created_at >= uuid_v7_to_ts('018c1ecb-d3b7-75b1-add9-62878b5152c7')
  order by created_at desc limit 10;
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.29..1.17 rows=10 width=41)
   ->  Custom Scan (ChunkAppend) on my_table  (cost=0.29..11.49 rows=126 width=41)
         Order: my_table.created_at DESC
         ->  Index Scan using _hyper_5_16_chunk_my_table_created_at_idx on _hyper_5_16_chunk  (cost=0.29..11.49 rows=126 width=41)
               Index Cond: (created_at >= '2023-11-30 05:55:23.703+00'::timestamp with time zone)
(5 rows)
```

– partition pruning in play, although it will require certain effort to have it in various queries. But it works.


--------

## Postscript

Also read the following comment by [@jamessewell](https://twitter.com/jamessewell), originaly posted
[here](https://twitter.com/jamessewell/status/1730125437903450129):

> If update your `create_hypertable` call with:
>
> ```
> time_column_name => 'id'
> time_partitioning_func => 'uuid_v7_to_ts'
> ```
>
> Then you'll be able to drop the `uuid_ts` col and your trigger!
>
> ```sql
> SELECT * FROM my_table WHERE id = '018c1ecb-d3b7-75b1-add9-62878b5152c7';
> ```
>
> Will just work 🪄
