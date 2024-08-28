Originally from: [tweet](https://twitter.com/samokhvalov/status/1722155043569496347), [LinkedIn post]().

---

# How to format SQL (SQL style guide)

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

There are various approaches to formatting SQL. There two major ones are:

1. **Traditional:** all keywords are uppercase, formatting is organized with a central whitespace pipe, advertised in Joe
   Celko’s "SQL Programming Style" book. One version is provided at [SQL Style Guide](https://www.sqlstyle.guide/).
   Example:

   ```sql
   SELECT a.title, a.release_date, a.recording_date
     FROM albums AS a
    WHERE a.title = 'Charcoal Lane'
       OR a.title = 'The New Danger';
   ```

   I personally find this style very old-fashioned, inconvenient, and cumbersome, and I prefer another one:

2. **Modern:** often (not always) lowercase keywords and left-align hierarchical indentation similar to what is used in
   regular programming languages. Example:

   ```sql
   select
       a.title,
       a.release_date,
       a.recording_date
   from albums as a
   where
       a.title = 'Charcoal Lane'
       or a.title = 'The New Danger';
   ```

Here, we'll discuss a version of the latter ("modern") approach, derived from
[Mozilla's SQL Style Guide](https://docs.telemetry.mozilla.org/concepts/sql_style.html).

The key difference from Mozilla's guide is that here we use lowercase SQL. The reason behind it is simple: an uppercase
SQL makes more sense when it's embedded into a different language, to help distinguish it from the surrounding code.

But if you work with large query texts provided separately in various SQL editors, perform optimization tasks, and so
on, the use of uppercase keywords becomes less convenient. Lowercase keywords are easier to type. Finally, constantly
shouting at your database is not a good idea.

At the same time, it might still make sense to use uppercase SQL keywords when using them in, for example, natural
language texts, for example:

> .. using multiple JOINs ..

## 1) Reserved words

Always use lowercase for reserved keywords such as `select`, `where`, or `as`.

## 2) Variable names

1. Use consistent and descriptive identifiers and names.
2. Use lower case names with underscores, such as `first_name`. **Do not use CamelCase.**
3. Functions such as `cardinality()`, `array_agg()`, and `substr()` are
   [identifiers](https://postgresql.org/docs/current/static/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS) and should
   be treated like variable names.
4. Names must begin with a letter and may not end in an underscore.
5. Only use letters, numbers, and underscores in variable names.

## 3) Be explicit

When choosing between explicit or implicit syntax, prefer explicit.

### 3.1) Aliasing

Always include the `as` keyword when aliasing a variable or table name, it's easier to read when explicit.

✅ Good:

```sql
select date(t.created_at) as day
from telemetry as t
limit 10;
```

❌ Bad:

```sql
select date(t.created_at) day
from telemetry t
limit 10;
```

### 3.2) JOINs

Always include the `join` type rather than relying on the default join.

✅ Good:

```sql
select
    submission_date,
    experiment.key as experiment_id,
    experiment.value as experiment_branch,
    count(*) as count
from
    telemetry.clients_daily
cross join
    unnest(experiments.key_value) as experiment
where
    submission_date > '2019-07-01'
    and sample_id = '10'
group by
    submission_date,
    experiment_id,
    experiment_branch;
```

❌ Bad:

```sql
select
    submission_date,
    experiment.key as experiment_id,
    experiment.value as experiment_branch,
    count(*) as count
from
    telemetry.clients_daily,
    unnest(key_value) as experiment -- implicit JOIN
where
    submission_date > '2019-07-01'
    and sample_id = '10'
group by
    1, 2, 3; -- implicit grouping column names
```

### 3.3) Grouping columns

Avoid using implicit grouping column names.

✅ Good:

```sql
select state, backend_type, count(*)
from pg_stat_activity
group by state, backend_type
order by state, backend_type;
```

🆗 Acceptable:

```sql
select
    date_trunc('minute', xact_start) as xs_minute,
    count(*)
from pg_stat_activity
group by 1
order by 1;
```

❌ Bad:

```sql
select state, backend_type, count(*)
from pg_stat_activity
group by 1, 2
order by 1, 2;
```

## 4) Left-align root keywords

All root keywords should start on the same character boundary.

✅ Good:

```sql
select
    client_id,
    submission_date
from main_summary
where
    sample_id = '42'
    and submission_date > '20180101'
limit 10;
```

❌ Bad:

```sql
  select client_id,
         submission_date
    from main_summary
   where sample_id = '42'
     and submission_date > '20180101';
```

## 5) Code blocks

Root keywords should be on their own line in all cases except when followed by only one dependent word. If there are
more than one dependent words, they should form a column that is left-aligned and indented to the left of the root
keyword.

✅ Good:

```sql
select
    client_id,
    submission_date
from main_summary
where
    submission_date > '20180101'
    and sample_id = '42'
limit 10;
```

It's acceptable to include an argument on the same line as the root keyword if there is exactly one argument.

🆗 Acceptable:

```sql
select
    client_id,
    submission_date
from main_summary
where
    submission_date > '20180101'
    and sample_id = '42'
limit 10;
```

❌ Bad:

```sql
select client_id, submission_date
from main_summary
where submission_date > '20180101' and (sample_id = '42' or sample_id is null)
limit 10;
```

❌ Bad:

```sql
select
    client_id,
    submission_date
from main_summary
where submission_date > '20180101'
    and sample_id = '42'
limit 10;
```

## 6) Parentheses

If parentheses span multiple lines:

1. The opening parenthesis should end the line.
2. The closing parenthesis should line up under the first character of the line that starts the multi-line construct.
3. Indent the contents of the parentheses one level.

✅ Good:

```sql
with sample as (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42'
  )
  ...
```

❌ Bad (terminating parenthesis on shared line):

```sql
with sample as (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42')
  ...
```

❌ Bad (no indent):

```sql
with sample as (
select
    client_id,
    submission_date
from main_summary
where
    sample_id = '42'
)
  ...
```

## 7) Boolean operators at the beginning of line

"and" and "or" should always be at the beginning of the line.

✅ Good:

```sql
...
where
    submission_date > '2018-01-01'
    and sample_id = '42'
```

❌ Bad:

```sql
...
where
    submission_date > '2018-01-01' and
    sample_id = '42'
```

## 8) Nested queries

Do not use nested queries. Instead, use common table expressions to improve readability.

✅ Good:

```sql
with sample as (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42'
)
select *
from sample
limit 10;
```

❌ Bad:

```sql
select *
from (
    select
        client_id,
        submission_date
    from main_summary
from main_summaryain_summary
    where
        sample_id = '42'
)
limit 10;
```

## About this document

This document is heavily influenced by the [SQL Style Guide](https://www.sqlstyle.guide/)
and [Mozilla SQL Style Guide](https://docs.telemetry.mozilla.org/concepts/sql_style.html). Proposals, extensions, and
fixes are welcome.
