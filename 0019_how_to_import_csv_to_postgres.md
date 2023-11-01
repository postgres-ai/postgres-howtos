Originally from: [tweet](https://twitter.com/samokhvalov/status/1713392789462175921), [LinkedIn post](...).

---

## Source CSV we'll be using as an example

In the following import examples, we're going to use the following snippet, gathering information about long-running
transactions from Postgres, into a CSV file:

```bash
header='header'

while sleep 1; do
  psql -XAtc "
      copy (
        with samples as (
          select
            clock_timestamp(),
            clock_timestamp() - xact_start as xact_duration,
            *
          from pg_stat_activity
        )
        select *
        from samples
        where xact_duration > interval '1 minute'
        order by xact_duration desc
      ) to stdout delimiter ',' csv ${header}
    " \
  2> >(ts | tee -a long_tx_$(date +%Y%m%d).err.log) \
  | tee -a long_tx_$(date +%Y%m%d).csv

  header=''
done
```

This is a modified version of the snippet from
the [Ad-hoc monitoring how-to](https://twitter.com/samokhvalov/status/1710510207875559844), with these adjustments:

- Write errors to a separate file (`***.err.log`) and prefix each line in this file with a timestamp. For regular
  records (`***.csv`), we don't want timestamp prefixes to break the CSV format – and these records are going to have
  timestamps generated on Postgres side
  anyway, `with clock_timestamp()`.
- In the CSV file, we want to have a header, but don't want it to be repeated.

## Method 1: Import a CSV snapshot to a regular table

To import a snapshot of the CSV file to Postgres, we need to create a table. There are two options here.

**Option 1:** exact table definition. If we know exactly (and in our case, we do) the structure that table needs to
have, we can just use this knowledge – I usually use the original query with `LIMIT 0` (optionally omitting `WHERE`
and `ORDER BY` clauses):

```sql
create table slow_tx_from_csv_1 as
with samples as (
select
  clock_timestamp(),
  clock_timestamp() - xact_start as xact_duration,
  *
from pg_stat_activity
)
select *
from samples
limit 0;
```

(Instead of `LIMIT 0`, you can also use `WITH NO DATA`, but I personally often forget about this syntax.)

This option provides a well-defined table structure (column types match the original structure being saved in CSV), but
it might not work well in all cases (e.g., in cases where we don't know how exactly the CSV was created).

**Option 2:** arbitrary structure with text columns. This approach is very flexible and allows dealing with the cases
where we don't know the structure in advance or just don't want to spend time to figure it out. In this case, we use the
fact that the first line of the CSV file is the header, and we create a table consisting of "text" columns only:

```bash
psql -c "create table slow_tx_from_csv_2($(
 head -n 1 long_tx_$(date +%Y%m%d).log.csv \
 | tr '[:upper:]' '[:lower:]' \
 | sed -e 's/[^,a-zA-Z0-9\r]/_/g' \
 | sed -e 's/,/ text, /g' \
) text)"
```

Now we can import a snapshot of the current CSV content to both files (here is an example for `slow_tx_from_csv_2`):

```bash
psql -c "copy slow_tx_from_csv_2 from '$(pwd)/long_tx_$(date +%Y%m%d).csv' delimiter ',' csv header"
```

Note that here we use SQL command COPY that works on the server side and requires the full path to the CSV file located
on the server where Postgres is running. There is also psql's command `\copy` that allows importing CSV file located on
the client's side (if these sides are different, in our case they are not, so we could use the either command). Docs:

- [server-side COPY](https://postgresql.org/docs/current/sql-copy.html)
- [psql's \copy](https://postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMANDS-COPY)

Note that when working with the "flexible" table format (text-only columns) – `slow_tx_from_csv_2` here – we need not
forget about type conversion. For example:

```
nik=# select clock_timestamp, pid, query from slow_tx_from_csv_2 order by clock_timestamp::timestamptz desc limit 2;
       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:15:27.984335-07 | 22957 | begin;
2023-10-14 19:15:26.933159-07 | 22957 | begin;
(2 rows)
```

This is not needed in the case of well-defined table structure, `slow_tx_from_csv_1`:

```
nik=# select clock_timestamp, pid, query from slow_tx_from_csv_1 order by clock_timestamp desc limit 2;
       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:15:19.567062-07 | 22957 | begin;
2023-10-14 19:15:18.515422-07 | 22957 | begin;
(2 rows)
```

## Method 2: Query CSV data live via file_fdw

This method should be used when we need to query the CSV file data via SQL "live" without loading a snapshot. To achieve
this, we'll be using [file_fwd](https://postgresql.org/docs/current/file-fdw.html).

Having a great advantage (live data!), this method has its obvious disadvantages:

- It's read-only (although, you can easily create a snapshot using `create table as select from ...`).
- The data is not protected by backup system you (hopefully) have, the storage is not reliable (a file).
- Performance limitations: you cannot create indexes to speed up the queries, so it doesn't work really well for huge
  data volumes (although, again, you can create a snapshot or a materialized view, and have indexes there).
- Some `COPY` options are not supported by `file_fdw`, such as the `FORCE_QUOTE` option.
- `file_fdw` is N/A on certain managed Postgres services (e.g., RDS).
- To work with `file_fdw`, a superuser access or having the privileges of the `pg_read_server_files` role is required.

First, prepare `file_fdw`:

```
create extension file_fdw;
create server "import" foreign data wrapper file_fdw;
```

Now, we're going to prepare the table (using the "flexible but text-only columns" approach as above). But this time it's
not a regular table – rather, it's a "foreign" table that provides a read-only SQL interface to the CSV file:

```bash
psql -c "create foreign table slow_tx_csv_live($(
 head -n 1 long_tx_$(date +%Y%m%d).csv \
 | tr '[:upper:]' '[:lower:]' \
 | sed -e 's/[^,a-zA-Z0-9\r]/_/g' \
 | sed -e 's/,/ text, /g'
) text) server import options (
 filename '$(pwd)/long_tx_$(date +%Y%m%d).csv',
 format 'csv',
 header 'on'
)"
```

That's it. We don't need to load anything, the data "is" already there – and we can watch how it changes live (not
forgetting about the type conversion):

```
nik=# select clock_timestamp, pid, query
from slow_tx_csv_live
order by clock_timestamp::timestamptz desc
limit 2 \watch 1
     Sat Oct 14 19:48:20 2023 (every 1s)

       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:48:19.598753-07 | 22957 | begin;
2023-10-14 19:48:18.554661-07 | 22957 | begin;
(2 rows)

     Sat Oct 14 19:48:21 2023 (every 1s)

       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:48:20.65133-07  | 22957 | begin;
2023-10-14 19:48:19.598753-07 | 22957 | begin;
(2 rows)

     Sat Oct 14 19:48:22 2023 (every 1s)

       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:48:21.706331-07 | 22957 | begin;
2023-10-14 19:48:20.65133-07  | 22957 | begin;
(2 rows)
```

## Bonus example (for bored)

If you're too bored with Postgres-only examples, here is my old example with the pokemon data obtained from a
[publicly available Google Spreadsheet](https://gist.github.com/NikolayS/a819f139c37e0d54ad4a4ca70764f225):

1) Install `file_fdw` in your database (run in psql):
   ```sql
   create extension file_fdw;
   create server import foreign data wrapper file_fdw;
   ```

2) Download as a CSV file:
   ```bash
   wget -O pokemons.csv \
   "https://docs.google.com/spreadsheets/d/14mIpk_ceBWVnjc1cPAD7AWeXkE8-t729pcLqwcX_Iik/export?format=csv"
   ```

3) Create a foreign table, defining its structure as a set of text-only columns based on the first line of the CSV
   file (header) and connect it to the CSV file:

   ```bash
   psql -c "create foreign table pokemon_imported($(
    head -n 1 pokemons.csv \
    | tr '[:upper:]' '[:lower:]' \
    | sed -e 's/[^,a-zA-Z0-9\r]/_/g' \
    | sed -e 's/,/ text, /g'
   ) text) server import options (
    filename '$(pwd)/pokemons.csv',
    format 'csv',
    header 'on'
   )"
   ```

4) Now it should work; test it (in psql):

   ```bash
   nik=# select generation, pokemon, type_i, max_hp
   from pokemon_imported
   order by max_hp::int desc
   limit 5;
   generation |  pokemon  | type_i  | max_hp
   ------------+-----------+---------+--------
   2          | Blissey   | Normal  | 415
   1          | Chansey   | Normal  | 407
   2          | Wobbuffet | Psychic | 312
   3          | Wailord   | Water   | 281
   5          | Alomomola | Water   | 273
   (5 rows)
   ```
