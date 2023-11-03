Originally from: [tweet](https://twitter.com/samokhvalov/status/1714153676212949355), [LinkedIn post]().

---

# How to set application_name without extra queries

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

`application_name` is useful to control what you and others is going to see in `pg_stat_activity` (it has a column with
the same name), and various tools that use this system view. Additionally, it appears in Postgres log (when `%a` is
included in `log_line_prefix`).

Docs: [application_name](https://postgresql.org/docs/current/runtime-config-logging.html#GUC-APPLICATION-NAME).

It's good practice to set `application_name` – for example, it can be very helpful for root-cause analysis during and
after incidents.

The approaches below can be used to set up other settings (including regular Postgres parameters such as
`statement_timeout` or `work_mem`), however here we'll focus on `application_name` particularly.

Typically, `application_name` is set via `SET` (apologies for the tautology):

```
nik=# show application_name;
 application_name
------------------
 psql
(1 row)

nik=# set application_name = 'human_here';
SET

nik=# select application_name, pid from pg_stat_activity where pid = pg_backend_pid() \gx
-[ RECORD 1 ]----+-----------
application_name | human_here
pid              | 93285
```

However, having additional query – even a blazing fast one – means an extra RTT 
([round-trip time](https://en.wikipedia.org/wiki/Round-trip_delay)), affecting latency, especially when communicating 
with a distant server.

To avoid it, use `libpq`'s options.

## Method 1: via environment variable

```bash
❯ PGAPPNAME=myapp1 psql \
  -Xc "show application_name"
 application_name
------------------
 myapp1
(1 row)
```

(`-X` means `ignore .psqlrc`, which is a good practice for automation scripts involving `psql`.)

## Method 2: throught connection URI

```bash
❯ psql \
  "postgresql://?application_name=myapp2" \
  -Xc "show application_name"
 application_name
------------------
 myapp2
(1 row)
```

The URI method takes precedence over `PGAPPNAME`.

## In application code

The described methods can be used not only with psql. Node.js example:

```bash
❯ node -e "
 const { Client } = require('pg');

 const client = new Client({
   connectionString: 'postgresql://?application_name=mynodeapp'
 });

 client.connect()
       .then(() => client.query('show application_name'))
       .then(res => {
           console.log(res.rows[0].application_name);
           client.end();
       });
 "
 mynodeapp
```
