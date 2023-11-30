Originally from: [tweet](https://twitter.com/samokhvalov/status/1729321940496924977), [LinkedIn post]().

---

# How to help others

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Two key principles of helping others (it can be internal help to your teammates, or external consulting, doesn't
matter):

1. Rely on good sources.
2. Test everything.

## Principle 1. Good sources

Rely on high-quality sources that you trust. For example, my point of view, ranking the quality/trustworthiness level,
roughly:

- PG docs – 9/10
- PG source code – 10/10 (source of truth!)
- StackOverflow random answers – 5/10 or lower (excluding
  [Erwin Brandstetter](https://stackoverflow.com/users/939860/erwin-brandstetter), [Laurenz Albe](https://stackoverflow.com/users/6464308/laurenz-albe), [Peter Eisentraut](https://stackoverflow.com/users/98530/peter-eisentraut) –
  these
  guys rock when they answer there, 8/10 or higher)
- the same for random blog posts
- both "Internals" books by [Suzuki](https://www.interdb.jp/pg/)
  and [Rogov](https://postgrespro.com/blog/pgsql/5969637) – 8/10 or higher
- etc.

And (important!) always provide the link to your sources; you get two benefits from this:

- advertise good source and pay them back;
- share the responsibility to some extent (very helpful if you are not very experienced yet; everyone might make a
  mistake).

## Principle 2. Verification – database experiments

Always doubt everything, don't trust language models, regardless of their nature.

If someone (including you or me) says something without verification via an experiment (test), it needs to be fixed –
using an experiment.

All decisions should be made based on data – the reliable data is gathered via testing.

Most database-related ideas are to be verified using database experiments.

Two types of database experiments:

1. Multi-session experiments – full-fledged benchmarks like those conducted using `pgbench`, JMeter, `sysbench`,
   `pgreplay-go`, etc.

   They aim to study the behavior of Postgres as a whole, all its components, and must be performed on dedicated
   resources where nobody else is doing any work. Environment should match production (VM and disk type, PG version,
   settings).

   Examples of such experiments include load testing, stress testing, performance regression testing. Main tools are
   those that aim to study macro-level query analysis: `pgss`, wait event analysis (aka active session history or
   performance/query insights), `auto_explain`, `pgBadger`, etc.

   More about this type of experiments: [Day 13: How to benchmark](0013_how_to_benchmark.md).

2. Single-session experiments – testing one or a sequence of SQL queries using a single session (sometimes, two), to
   check query syntax, study individual query behavior, optimize particular query, etc.

   These experiments can be conducted in shared environments, on weaker machines. However, to study query performance,
   you need to have the same PG version, same or similar database, and matching planner settings (how to do it:
   [Day 56: How to make the non-production Postgres planner behave like in production](0056_how_to_imitate_production_planner.md)).
   In this case, it should be kept in mind that
   timing metrics might be significantly off compared to production, and the main attention in the query optimization
   process should be paid to execution plans and data volumes (actual rows, buffer operation counts provided by the
   option `BUFFERS` in `EXPLAIN`).

   Examples of such experiments: checking syntax and logical behavior of a sequence of SQL queries, query performance
   analysis `EXPLAIN (ANALYZE, BUFFERS)` and testing optimization ideas, schema change testing, etc. It is very helpful
   to be able to quickly clone large databases not paying extra money for storage (Neon, Amazon Aurora). And it's even
   better if no extra money is paid for both storage and compute, this truly unlocks testing activities including
   automated tests in CI/CD (DBLab Engine [@Database_Lab](https://twitter.com/Database_Lab)).

## Summary

As you can see, the principles are extremely simple:

- read good papers, and
- don't blindly trust – test everything.
