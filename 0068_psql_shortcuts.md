Originally from: [tweet](https://twitter.com/samokhvalov/status/1731340441403215959), [LinkedIn post]().

---

# psql shortcuts

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

In my tool [postgres_dba](https://github.com/NikolayS/postgres_dba/), the setup instructions have this:

```bash
printf "%s %s %s %s\n" \\set dba \'\\\\i $(pwd)/start.psql\' >> ~/.psqlrc
```

This provides a way to call `start.psql` by just typing `:dba` in psql, and this line works in both bash and zsh (which
is good because macOS switched to zsh as the default shell a few years ago, while bash is usually the default shell on
Linux distros).

One can easily add their own instructions to `.psqlrc`, defining various handy shortcuts. For example:

```sql
\set pid 'select pg_backend_pid() as pid;'
\set prim 'select not pg_is_in_recovery() as is_primary;'
\set a 'select state, count(*) from pg_stat_activity where pid <> pg_backend_pid() group by 1 order by 1;'
```

👉 This adds simple shortcuts:

1) Current PID:

    ```sql
    nik=# :pid
      pid
    --------
     513553
    (1 row)
    ```

2) Is this not a primary?

    ```sql
    nik=# :prim
     is_primary
    ------------
     t
    (1 row)
    ```

3) Simple activity summary

    ```sql
    nik=# :a
            state        | count
    ---------------------+-------
     active              |    19
     idle                |   193
     idle in transaction |     2
                         |     7
    (4 rows)
    ```

The value of a currently set psql variable can be passed in an SQL context as a string using this interesting syntax:

```sql
nik=# select :'pid';
            ?column?
---------------------------------
 select pg_backend_pid() as pid;
(1 row)
```

It is important not to forget to use the option `-X` (` --no-psqlrc`) in scripting, so nothing that you put in `.psqlrc`
would affect the logic of your scripts.
