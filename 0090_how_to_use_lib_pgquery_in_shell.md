Originally from: [tweet](https://twitter.com/samokhvalov/status/1740256615549599764), [LinkedIn post]().

---

# How to use lib_pgquery in shell to normalize and match queries from various sources

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## Why use lib_pgquery

In [Day 12: How to find query examples for problematic pg_stat_statements records](0012_from_pgss_to_explain__how_to_find_query_examples.md)
it was mentioned that query normalization can be done using [lib_pgquery](https://github.com/pganalyze/libpg_query). This
library builds a tree representation of query text, and also computes so-called "fingerprint" – a hash of the normalized
form of the query (query text where all parameters are removed).

This is helpful in various cases, for example:

- If you need to match normalized queries in `pg_stat_statements` with individual query texts from `pg_stat_activity` or
  Postgres logs, and you use Postgres version older than 14, where
  [compute_query_id](https://postgresqlco.nf/doc/en/param/compute_query_id/) was implemented to solve this problem.
- If you use newer version of Postgres, but `compute_query_id` is `off`.
- If you use query texts from different sources and/or are unsure that the standard `query_id` (aka "`queryid"` - the
  naming is not unified across tables) can be a reliable way of matching.

The basic `lib_pgquery` is written in C, and it is used in libraries for various languages:

- [Ruby](https://github.com/pganalyze/pg_query)
- [Python](https://pypi.org/project/pglast/)
- [Go](https://github.com/pganalyze/pg_query_go)
- [Node](https://github.com/pyramation/pgsql-parser)

## Docker version for CLI use

For convenience, my colleagues took the Go version and wrapped it into docker, allowing its use in CLI style, in shell:

```
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = 123;'
{
  "query": "select c1 from t1 where c2 = 123;",
  "normalizedQuery": "select c1 from t1 where c2 = $1;",
  "fingerprint": "0212acd45326d8972d886d4b3669a90be9dd4a9853",
  "tree": [...]
  ]
}
```

– it produces a JSON.

To get the normalized query value, we can use `jq`:

```
❯ docker run --rm postgresai/pg-query-normalize \
      'select c1 from t1 where c2 = 123;' \
    | jq -r '.normalizedQuery'
select c1 from t1 where c2 = $1;
```

To extract just the fingerprint as text:

```
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = 123;' \
  | jq -r '.fingerprint'
0212acd45326d8972d886d4b3669a90be9dd4a9853
```

If we use different parameters, fingerprint doesn't change – let's use 0 instead of 123:

```
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = 0;' \
  | jq -r '.fingerprint'
0212acd45326d8972d886d4b3669a90be9dd4a9853
```

## Normalized queries as input

And – this is a very good property! – if a query is already normalized, and we have placeholders (`$1`, `$2`, etc.) instead
of parameters, the value remains the same:

```
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = $1;' \
  | jq -r '.fingerprint'
0212acd45326d8972d886d4b3669a90be9dd4a9853
```

But it changes if the query is different – say, the `SELECT` clause is different, here we use `select *` instead of
`select c1`, resulting in a new fingerprint value:

```
❯ docker run --rm postgresai/pg-query-normalize \
    'select * from t1 where c2 = $1;' \
  | jq -r '.fingerprint'
0293106c74feb862c398e267f188f071ffe85a30dd
```

## IN lists

Finally, unlike the in-core `queryid`, `lib_pgquery` ignores the number of parameters in the `IN` clause – this behavior suits
better for query normalization and matching. Compare:

```
❯ docker run --rm postgresai/pg-query-normalize \
    'select * from t1 where c2 in (1, 2, 3);' \
  | jq -r '.fingerprint'
022fad3ad8fab1b289076f4cfd6ef0a21a15d01386

❯ docker run --rm postgresai/pg-query-normalize \
  'select * from t1 where c2 in (1000);' \
| jq -r '.fingerprint'
022fad3ad8fab1b289076f4cfd6ef0a21a15d01386
```
