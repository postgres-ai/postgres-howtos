Originally from: [tweet](https://twitter.com/samokhvalov/status/), [LinkedIn post]().

---

# Postgres major upgrade without any downtime for a very large cluster running under heavy load

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!


Right now on HN frontpage there is an article by knock.app
on [zero downtime Postgres upgrades](https://news.ycombinator.com/item?id=38616181).

Later this week, [Alexander Sosna](https://twitter.com/xxorde) (GitLab) is
presenting [a talk](https://postgresql.eu/events/pgconfeu2023/schedule/session/4791-how-we-execute-postgresql-major-upgrades-at-gitlab-with-zero-downtime/)
explaining how GitLab's large clusters were upgraded under heavy load without any downtime – I highly recommend looking
that that work

There are many details behind the proper zero downtime upgrade, many challenges to solve, and here I present only a
high-level plan. It works well for very large (dozens of TiB) clusters, with many replicas, and working under high TPS
(dozens of 10k). The process described here involves logical replication (with the "physical-to-logical" trick) and
PgBouncer's PAUSE/RESUME (assuming that PgBouncer is used).

The detailed material will require several separate howtos to be written.

The process consists of 2 steps:

1) **UPGRADE:** New cluster is created, running on new Postgres major version.

2) **SWITCHOVER:** step by step switchover of the traffic.

## Step 1: UPGRADE

There is a "standard" approach when a new cluster is first created from scratch (`initdb`) and then a brand new logical
replica is created based on it (`copy_data = false` when creating logical subscription). However, this way of logical
replica provisioning takes a lot of time can be very problematic to execute under heavy load.

An alternative is to take a physical replica and convert it to logical. This is relatively easy to do knowing the
logical slot's LSN position and using `recovery_target_lsn`. This recipe is called "physical2logical" conversion, and it
allows for the creation of a logical replica based on physical replica (e.g. created based on a cloud snapshot in
minutes), reliably and quickly.

However, combining the physical2logical conversion with `pg_upgrade` should be done with care.
Details can be found [here](https://postgresql.org/message-id/flat/20230217075433.u5mjly4d5cr4hcfe%40jrouhaud).

Steps:

1. Create a new cluster, which will be a secondary cluster using cascaded physical replication (Patroni supports it).

2. Ensure that the new cluster's leader is not significantly lagging behind the old cluster's primary.

3. Stop new cluster's node in proper order (making sure stopped replicas are fully in sync with their leader).

4. On the old cluster's primary, create a logical slot and publication `FOR ALL TABLES` (this doesn't require
   table-level locks, thus we don't need low `lock_timeout` and retries) and remember the slot's position.

5. Reconfigure new cluster's leader, setting `recovery_target_lsn` to the remembered slot's LSN, and disabling
   `restore_command`.

6. Start new cluster's leader and replicas, let them reach the desired LSN. Then the leader can be promoted.

7. Stop new cluster's nodes again, in proper order.

8. Run `pg_upgrade --link` on the new cluster's leader.

9. Use `rsync --hard-links --size-only` on the new cluster's replicas – this is disputable step (see
   details [here](https://postgresql.org/message-id/flat/CAM527d8heqkjG5VrvjU3Xjsqxg41ufUyabD9QZccdAxnpbRH-Q%40mail.gmail.com)),
   but this is what most people use for in-place (w/o logical replication) upgrades with `pg_upgrade --link`, and there
   is no another fast alternative invented yet.

10. Configure new cluster's leader (now primary already) to use logical replication – create subscription,
    with `copy_data = false` and let it catch up with the working old cluster.

During all these steps the old cluster is up and running, and new cluster is invisible to users. This gives you a huge
benefit of testing the whole process right in production (after proper testing in lower environments).

## Step 2: SWITCHOVER

First, it makes sense to switch over the read-only (RO) traffic. If the application allows, it makes sense to first
redirect only part of the RO traffic to new replicas. This would require an advanced replication lag detection in the
load balancing code
(see:
[Day 17: How to determine the replication lag - Hybrid case: logical & physical](./0017_how_to_determine_the_replication_lag.md#hybrid-case-logical-physical)).

When it is time to redirect the RW traffic, to achieve zero downtime, one can use PgBouncer's PAUSE/RESUME. If there are
multiple PgBouncer nodes (running on separate hosts/ports, or involving `SO_REUSEPORT`), it is important to implement a
graceful approach for PAUSE acquisition. Once all PAUSEs are acquired, the traffic can be redirected, and it is time to
issue RESUME. Before that it is important to make sure that all writes are fully propagated to the new primary.

It is important to have measures developed to protect the old cluster from writes coming from application or users –
e.g. adjusting `pg_hba.conf` or shutting the cluster down. However, for advanced rollback capabilities, it makes sense to
implement "reverse" logical replication, in the same manner as "forward" one, setting it up during switchover. In this
case, the writes to the old cluster are allowed – but only from logical replication. The reverse replication allows for
rollback even some time after the whole process is finished, this makes the whole process fully reversible from any
point.

As already mentioned, there are many aspects to address, this is only a high-level plan. If you can attend this talk, do
it [here](https://postgresql.eu/events/pgconfeu2023/schedule/session/4791-how-we-execute-postgresql-major-upgrades-at-gitlab-with-zero-downtime/).
