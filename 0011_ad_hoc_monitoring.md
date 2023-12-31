Originally from: [tweet](https://twitter.com/samokhvalov/status/1710176204953919574), [LinkedIn post](...).

---

# Ad-hoc monitoring

<img src="files/0011_cover.png" width="600" />

// I post a new PostgreSQL "howto" article every day. Join me in this journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

In some cases, we need to observe some values in Postgres or the environment it's running on (OS, FS, etc.), and:
- we don't have good monitoring at all, or
- the existing monitoring is lacking something we need, or
- we don't fully trust the existing monitoring (e.g., you're an engineer who has only started working with some system).

Being able to organize an "ad hoc" observation on specific metrics is an important skill to master. Here we'll describe some tips and an approach that you can find useful.

In some cases, you can quickly install tools like [Netdata](https://netdata.cloud) (a very powerful modern monitoring that can be quickly installed, has a Postgres plugin or ad-hoc console tools such as [pgCenter](https://github.com/lesovsky/pgcenter) or [pg_top](https://pg_top.gitlab.io). But you still may want to monitor some specific aspects manually.

Below, we assume that we are using Linux, but most of the considerations can be applied to macOS or BSD systems as well.

## Key principles
- observation should not depend on your internet connection
- save results for long-term storage and later analysis
- know timestamps
- protect from accidental program interruption (`Ctrl-C`)
- sustain Postgres restarts
- observe both normal messages and errors
- prefer collecting data in a form useful for programmed processing (e.g., CSV)

## An example
Let's assume we need to collect samples `pg_stat_activity` (`pgsa`) to study long-running transactions – those transactions that last longer than 1 minute.

Here is the recipe – and below we discuss it in detail.

1. Start a `tmux` session. Here is an idempotent snippet that you can use – it either connects to existing session called "observe" or creates it, if it's not found:
    ```shell
    export TMUX_NAME=observe
    tmux a -t $TMUX_NAME || tmux new -s $TMUX_NAME
    ```

2. Now, start collecting `pgsa` samples in a log every second infinitely (until we interrupt this process):
    ```shell
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
            ) to stdout delimiter ',' csv
        " 2>&1 \
        | tee -a long_tx_$(date +%Y%m%d).log.csv
    done
    ```

Below we discuss various tricks and tips used here.

## Use tmux (or screen)
The benefits here are straightforward: if you have internet connectivity issues and disconnected, then work is not lost if `tmux` session is running on a server. Following a predefined session naming convention can be helpful in case you work in a team.

## How to use loops / batches
Some programs you'll use support batched reporting (examples: `iostat -x 5`, `top -b -n 100 -d 5`), some don't support it. In the latter case, use a `while` loop.

I prefer using an infinite loop like `while sleep 5; do ... ; done` – this approach has a small downside – it starts with sleeping first, and only then perform useful work – but it has a benefit that most of the time, you can interrupt using `Ctrl-C`.

## Use psql options: -X, -A, -t

Best practices of using `psql` for observability-related sampling (and work automation in general):
1. Always use `-X` – this is going to be very helpful one day, when you happen to work with a server, when an unexpected `~/.psqlrc` file that contains something that could make `psql` output unpredictable (e.g., a simple `\timing on`). The option `-X` tells `psql` to ignore `~/.psqlrc`.
2. Options `-A` and `-t` – provide unaligned output without headers. This is helpful to have a result that is easier to parse and process in large volumes.

## Use CSV as output
There are several ways to produce a CSV:
- use psql's command `\copy`  – in this case, results will be saved to file on client's side
- `copy (...) to '/path/to/file'` – this will save results on server (there might be permissions issue if path is not writable for OS user under which Postgres is running)
- `psql --csv -F,` – produce a CSV (but there may be issues with escaping values collide with field separator)
- `copy (...) to stdout` – this approach is, perhaps, most convenient for the sampling/logging purposes

## Master log-fu: don't lose STDERR, have timestamps, append
For later analysis (who knows what we'll decide to check a couple of hours later?) it is better to save everything to a file.

But it is also critically important not to lose the errors – usually, they are printed to `STDERR`, so we either need to write them to a separate file. We also might want not to lose the existing content of the file so instead of just overwriting (`>`) we want to append (`>>`):
```shell
command   2>>error.log >>messages.log
```

Or just redirect everything to a single file:
```shell
command  &>>everything.log
```

If you want to both see everything and log it, use `tee` – or, with append mode, `tee -a` (here, `2>&1` redirects `STDERR` to `STDOUT` first, an then `tee` gets everything from `STDOUT`):
```shell
commend 2>&1 | tee -a everything.log
```

If the output you have lacks timestamps (not the case with the psql snippet we used above though), then use `ts` to prepend each line with a timestamp:
```shell
command 2>&1 \
  ts \
  tee -a everything.log
```

Finally, it is usually wise to name the file with result with some details and current date (or even date + time, depending on situation):
```shell
command 2>&1 \
  | ts \
  | tee -a observing_our_comand_$(date +%Y%m%d).log
```

One downside of using `tee` is that, in some cases, you might accidentally stop it (e.g., pressing `Ctrl-C` in a wrong `tmux` window/pane). Due to this, some people prefer using `nohup ... &` to run observability actions in background and observing the result using `tail -f`.

## Processing result
The result, if it's CSV, is easy to process later – Postgres suits really well for it. We just need to remember the set of columns we've used, so we can create a table and load the data:
```
nik=# create table log_loaded as select clock_timestamp(), clock_timestamp() - xact_start as xact_duration, * from pg_stat_activity limit 0;
SELECT 0
nik=# copy log_loaded from '/Users/nik/long_tx_20231006.log.csv' csv delimiter ',';
COPY 4
```

And then analyze it with SQL. Or you may want to analyze and visualize the data somehow – for instance, load the CSV to some spreadsheet tool and then analyze the data and create some charts there.

## Alternative sampling – right from psql
In psql, you can use `\o | tee -a logfile` and `\watch` to both see the data and log it at the same time – although, notice that it won't capture errors in the file. Example:
```
nik=# \o | tee -a alternative.log

nik=# \t
Tuples only is on.
nik=# \a
Output format is unaligned.
nik=# \f ┃
Field separator is "┃".
nik=# select clock_timestamp(), 'test' as c1 \watch 3
2023-10-06 21:01:05.33987-07┃test
2023-10-06 21:01:08.342017-07┃test
2023-10-06 21:01:11.339183-07┃test
^C
nik=# \! cat  alternative.log
2023-10-06 21:01:05.33987-07┃test
2023-10-06 21:01:08.342017-07┃test
2023-10-06 21:01:11.339183-07┃test
nik=#
```

---
