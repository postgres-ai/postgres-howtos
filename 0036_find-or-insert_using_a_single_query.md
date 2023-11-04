Originally from: [tweet](https://twitter.com/samokhvalov/status/1719635712545591630), [LinkedIn post]().

---

# "Find-or-insert" using a single query

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Consider the task: you need to read a row, and if it doesn't exist, insert it (and read what's just inserted). Using a
single query.

The table we'll be using below for examples:

```sql
create table t1 (
  id bigserial primary key,
  ts timestamptz not null unique
);
```

## Approach 1: INSERT ... CONFLICT

One might think `INSERT ... CONFLICT` or `MERGE` are good for this. No.

Let's see:

```sql
insert into t1(ts)
select now()::timestamptz(0)
on conflict (ts) do nothing
returning *
```

This won't return anything if a collision happens. Per the [doc](https://postgresql.org/docs/current/sql-insert.html): 

> Only rows that were successfully inserted or updated will be returned.

To fix it, we need to extend the query to have the `UPDATE`:

```sql
insert into t1(ts)
select now()::timestamptz(0)
on conflict (ts) do update set ts = excluded.ts
returning *
```

But this leads to performing an `UPDATE` where we need just a `SELECT`, and this is a huge overhead that we definitely
should avoid.

Another overhead that both queries have: each time a `ON CONFLICT` collision happens, we have a wasted sequence
increment (and this would be the same if we used `GENERATED ALWAYS AS IDENTITY`, because it uses sequences under the
hood).

> **TODO:** for the future edits of this howto: consider `MERGE` and explain why it's not good as well

## Approach 2: Naive CTE with UPDATE-or-SELECT

One might think this using CTE can help here:

```sql
with val(ts) as (
  values(now())
), inserted as (
  insert into t1(ts)
  select val.ts
  from val
  where not exists (
    select
    from t1
    where t1.ts = val.ts
  )
  returning *
)
select * from inserted
union all
select t1.* from t1 join val using (ts);
```

It does help, but only if you work alone with your database. In concurrent workloads, this is going to lead to
occasional rollbacks. To see it, let's use `now()::timestamptz(0)` instead of `now()`. So, every second we have only 1
possible value to be found-or-inserted, and let's run this query with 8 sessions at TPS=100:

```bash
echo "with val(ts) as (
  values(now()::timestamptz(0))
), inserted as (
  insert into t1(ts)
  select val.ts
  from val
  where not exists (
    select
    from t1
    where t1.ts = val.ts
  )
  returning *
)
select * from inserted
union all
select t1.* from t1 join val using (ts)" > upsert.sql

❯ pgbench -c8 -j8 -R100 -P10 -T30 -nr -fupsert.sql
pgbench (15.4 (Homebrew))
pgbench: error: client 0 script 0 aborted in command 0 query 0: ERROR:  duplicate key value violates unique constraint "t1_ts_key"
DETAIL:  Key (ts)=(2023-11-01 01:00:28-07) already exists.
```

Why errors can happen here? Because between checking the row with sub-`SELECT` and attempting to run an `INSERT` some
very brief time always exists, during which another session might perform the `INSERT`. So this approach is not working 
well in general case. But we can improve it.

## Approach 3: improved CTE

Here is an improved version that doesn't suffer from the issues described above:

```sql
with val(ts) as (
  values(now()::timestamptz(0))
), select_attempt as (
  select t1.*
  from t1
  join val using (ts)
), inserted as (
  insert into t1(ts)
  select val.ts
  from val
  where not exists (
    select from select_attempt
  )
  on conflict (ts) do nothing
  returning *
)
select * from select_attempt
union all -- "all" is for troubleshooting only
select * from inserted
```

Putting it to the file `improved.sql`, let's test it with `pgbench`, in the same way as the previous approach:

```bash
❯ pgbench -c8 -j8 -R100 -P10 -T30 -nr -fimproved.sql

pgbench (15.4 (Homebrew))
progress: 10.0 s, 104.1 tps, lat 3.666 ms stddev 1.777, 0 failed, lag 2.849 ms
progress: 20.0 s, 99.3 tps, lat 3.801 ms stddev 3.299, 0 failed, lag 2.852 ms
transaction type: improved.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 8
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 3046
number of failed transactions: 0 (0.000%)
latency average = 3.722 ms
latency stddev = 2.521 ms
rate limit schedule lag: avg 2.859 (max 27.065) ms
initial connection time = 22.667 ms
tps = 101.624575 (without initial connection time)
statement latencies in milliseconds and failures:
         0.847           0  with val(ts) as (
```

👉 No more failures. And no more wasted sequence values or "overkilling" `UPDATE`s (easy to check in `psql`, running the
query with `now()::timestamptz(0)` and `\watch .2`).


---

## 📝 Correction and additional notes

1. Occasional empty results in highly concurrent workloads

   `select * from select_attempt` in the last query may lead to empty results occasionally – to fix it, we need to re-read
   from the table. This is a problem. Replacing it with another read attempt won't help. 
   
   **Solution**: Use `ON CONFLICT DO UPDATE`, which here is acceptable since it comes after a `SELECT` attempt, and the 
   overhead discussed above hits us only rarely (at same frequency as query failures in the case of "naive CTE").
   
   Testing it:
   
   ```bash
   echo "with val(ts) as (
     values(now()::timestamptz(0))
   ), select_attempt as (
     select t1.*
     from t1
     join val using (ts)
   ), inserted as (
     insert into t1(ts)
     select ts from val
     except
     select ts from select_attempt
     on conflict (ts) do update set ts = excluded.ts
     returning *
   ), res as (
     select * from select_attempt
     union all -- "all" is for troubleshooting only
     select * from inserted
   )
   select 1/count(*) from res;
   " > improved.sql
   
   pgbench -c8 -j8 -R100 -P10 -T120 -nr -fimproved.sql
   ```

   👉 There are no `division by zero` errors (at least in my test).

2. Using `EXCEPT` instead of `WHERE NOT EXISTS` might look more attractive (no subqueries) – the idea
   from [@be_haki's Tweet](https://twitter.com/be_haki/status/1718993194938187935) and I like it!

3. I wish all this would be much easier.
