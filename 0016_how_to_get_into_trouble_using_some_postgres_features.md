Originally from: [tweet](https://twitter.com/samokhvalov/status/1712342522314572064), [LinkedIn post](...).

---

# How to get into trouble using some Postgres features

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Today we have quite entertaining material, but knowing (and avoiding) these things can save you time and effort.

## NULLs

`NULL`s, while being very common, are the most popular way to get into trouble when using SQL in general, and Postgres
is
no exception.

For example, one may forget that concatenation (`||`), arithmetic operations (`*`, `/`, `+`, `-`), traditional
comparison operators (`=`, `<`, `>`, `<=`, `>=`, `<>`) are all not NULL-safe operations, and later be very surprised
that the result is lost.

Especially it hurts when you build a startup and some important business logic depends on it, a query not
handling `NULL`s properly, leading to loss of user base or money or time (or all that):

```
nik=# \pset null ∅
Null display is "∅".

nik=# select null + 1;
?column?
  ----------

          ∅

(1 row)
```

`NULL`s can be really dangerous and even experienced engineers continue to bump into issues when working with them. Some
useful materials to educate yourself:

- [NULLs: the good, the bad, the ugly, and the unknown](https://postgres.fm/episodes/nulls-the-good-the-bad-the-ugly-and-the-unknown)
  (podcast)
- [What is the deal with NULLs?](http://thoughts.davisjeff.com/2009/08/02/what-is-the-deal-with-nulls/)

A couple of tips – how to make your code NULL-safe:

- Consider using expressions like `COALESCE(val, 0)` for replace `NULL`s with some value (usually `0` or `''`).
- For comparison, instead of `=` or `<>`: `IS [NOT] DISTINCT FROM` (check out the `EXPLAIN` plan though).
- Instead of concatenation, use: `format('%s %s', var1, var2)`.
- Don't use `WHERE NOT IN (SELECT ...)` – use `NOT EXISTS` instead (
  see thia [JOOQ blog post](https://jooq.org/doc/latest/manual/reference/dont-do-this/dont-do-this-sql-not-in/)).
- Just be careful. `NULL`s are treacherous.

## Subtransactions under heavy loads

If you aim to grow to dozens or hundreds of thousands of TPS and want to have various issues, use subtransactions.
Probably, you use them implicitly – e.g., if you use Django, Rails, or `BEGIN/EXCEPTION` blocks in PL/pgSQL.

Why you might want to get rid of subtransactions completely:
[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful)

## int4 PK

Zero-downtime conversion of `int4` (a.k.a. int a.k.a. integer) PK to `int8` when the table has 1B rows requires a lot of
efforts. While table `(id int4, created_at timestamptz)` is going to take the same disk space as
`(id int8, created_at timestamptz)` due to [alignment padding](https://stackoverflow.com/a/7431468/459391).

## (Exotic) SELECT INTO is not you think it is

One day I was debugging a PL/pgSQL function, and copy-pasted a query like this, to `psql`:

```
nik=# select * into var from t1 limit 1;
SELECT 1
```

It worked! This is a huge surprise – in SQL context (not
PL/pgSQL), [SELECT INTO](https://postgresql.org/docs/current/sql-selectinto.html) is a DDL command that creates a table
and inserts data into it (shouldn't this be deprecated already?)

## Thinking that "transactional DDL" is easy

Yes, Postgres has "transactional DDL" and you can benefit from it a lot – until you cannot. Under load, you cannot rely
on it – instead, you need to start using zero-downtime methodologies and avoid mistakes (
read: [common db schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes), and rely
on "non-transactional" DDL such as
`CREATE INDEX CONCURRENTLY`, assuming that some attempts might fail, after which cleanup is needed before retrying.

A big problem with DDL deployment under load is that by default, you can have downtime attempting to deploy a very light
schema change – unless you implement a logic with low `lock_timeout` and retries (
see: [zero-downtime postgres schema migrations lock timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)).

## DELETE a lot of rows with one command

This is a good way to get into trouble: issue a `DELETE` of millions of rows and wait. If `checkpointer` is not tuned
(`max_wal_size = 1GB`), if tuples are deleted via an `IndexScan` access (meaning the process of making pages dirty is
quite "random"), and disk IO is quite throttled, this may put your system down. And even if it survives the stress,
you'll get:

- risks of locking issues (`DELETE` blocking some writes issued by other users),
- a large number of dead tuples produced, to be converted to bloat later by `autovacuum`.

What to do:

- split to batches,
- if massive write is inevitable, consider raising `max_wal_size` temporarily, which doesn't require restart (however:
  this potentially increases recovery time if server crashes during this procedure).

Read [common db schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes#case-4-unlimited-massive-change).

## Other "Don't do" articles

- [Depesz: Don’t do these things in PostgreSQL](https://depesz.com/2020/01/28/dont-do-these-things-in-postgresql/)
- [PostgreSQL Wiki: Don't Do This](https://wiki.postgresql.org/wiki/Don't_Do_This)
- [JOOQ: Don't do this](https://jooq.org/doc/latest/manual/reference/dont-do-this/)
