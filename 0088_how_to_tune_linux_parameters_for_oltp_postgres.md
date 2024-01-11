Originally from: [tweet](https://twitter.com/samokhvalov/status/1739545313311133849), [LinkedIn post]().

---

# How to tune Linux parameters for OLTP Postgres

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Here are general recommendations for basic tuning of Linux to run Postgres under heavy OLTP (web/mobile apps) workloads.
Most of them are default settings used in [postgresql_cluster](https://github.com/vitabaks/postgresql_cluster).

Consider the parameters below as entry points for further study, and values provided as just rough tuning that is worth
reviewing for a particular situation.

Most of the parameters can be changed in `sysctl.conf`. After changing it, this needs to be called:

```bash
sysctl -p /etc/sysctl.conf
```

Temporary change (taking `vm.swappiness` as example):

```bash
sudo sysctl -w vm.swappiness=1
```

or:

```bash
echo 1 | sudo tee /proc/sys/vm/swappiness
```

## Memory management

1) `vm.overcommit_memory = 2`

    Avoid memory overallocation to prevent OOM killer from affecting Postgres.

2) `vm.swappiness = 1`

    Minimalistic swap, not fully switching it off. 
    > 💀 This is a controversial topic; I personally have used 0 here under
    heavy loads in mission-critical systems and taking my chances with the OOM killer; however, many experts suggest not
    turning it off completely and using a low value – 1 or 10.

    **Good articles on this topic:**

    - [Deep PostgreSQL Thoughts: The Linux Assassin](https://crunchydata.com/blog/deep-postgresql-thoughts-the-linux-assassin) 
      (2021; k8s context) by [@josepheconway](https://twitter.com/josepheconway)
    
    - [PostgreSQL load tuning on Red Hat Enterprise Linux](https://redhat.com/en/blog/postgresql-load-tuning-red-hat-enterprise-linux) (2022) 

3) `vm.min_free_kbytes = 102400`

    Ensure available memory for Postgres during memory allocation spikes.

4) `transparent_hugepage/enabled=never`, `transparent_hugepage/defrag=never`

    Disable Transparent Huge Pages (THP) as they can induce latency and fragmentation not suitable for Postgres OLTP
    workloads. Disabling THP is generally recommended for OLTP systems (e.g., Oracle).

    - [Ubuntu/Debian](https://stackoverflow.com/questions/44800633/how-to-disable-transparent-huge-pages-thp-in-ubuntu-16-04lts)
    - [Red Hat](https://access.redhat.com/solutions/46111)

## I/O Management

5) `vm.dirty_background_bytes = 67108864`

6) `vm.dirty_bytes = 536870912`

    These ^ two are to tune [pdflush](https://lwn.net/Articles/326552/) to prevent IO lag spikes. See
    also: [PgCookbook - a PostgreSQL documentation project](https://github.com/grayhemp/pgcookbook/blob/master/database_server_configuration.md) by
    [@grayhemp](https://twitter.com/grayhemp).

## Network Configuration

> 📝 note that below ipv4 settings are provided; 
> 🎯 **TODO:** ipv6 options

7) `net.ipv4.ip_local_port_range = 10000 65535`

    Allows handling of more client connections.

8) `net.core.netdev_max_backlog = 10000`

    Handles bursts of network traffic without packet loss.

9) `net.ipv4.tcp_max_syn_backlog = 8192`

    Accommodates high levels of concurrent connection attempts.

10) `net.core.somaxconn = 65535`

    Increases the limit for queued socket connections.

11) `net.ipv4.tcp_tw_reuse = 1`

    Reduces connection setup time for high throughput OLTP applications.

## NUMA Configuration

12) `vm.zone_reclaim_mode = 0`

    Avoids the performance impact of reclaiming memory across NUMA nodes for Postgres.

13) `kernel.numa_balancing = 0`

    Disables automatic NUMA balancing to enhance CPU cache efficiency for Postgres.

14) `kernel.sched_autogroup_enabled = 0`

    Improves process scheduling latency for Postgres.

## Filesystem and File Handling

15) `fs.file-max = 262144`

    Maximum number of file handles that the Linux kernel can allocate. When running a database server like Postgres, having
    enough file descriptors is critical to handle numerous connections and files simultaneously.

> 🎯 **TODO:** review and adjust for various popular OSs
