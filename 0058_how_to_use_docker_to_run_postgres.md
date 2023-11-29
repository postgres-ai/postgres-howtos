Originally from: [tweet](https://twitter.com/samokhvalov/status/1727705412072554585), [LinkedIn post]().

---

# How to use Docker to run Postgres

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

This howto is for users who use or need to use Postgres, but are not experienced in using Docker.

Running docker in container for development and testing can help you align the sets of libraries, extensions, software
versions between multiple environments.

## Docker installation – macOS

Installation using [Homebrew](https://brew.sh):

```bash
brew install docker docker-compose
```

## Docker installation – Ubuntu

```bash
sudo apt-get update
sudo apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository -y \
 "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update && sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  http://containerd.io \
  docker-compose-plugin
```

To avoid the need to use `sudo` to run `docker` commands:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

## Run Postgres in container with persistent PGDATA

Assuming we want the data directory (`PGDATA`) be in `~/pgdata` and container named as `pg16`:

```bash
sudo docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  postgres:16
```

## Check logs

Last 5 minutes of logs, with timestamps, and observing new coming log entries:

```bash
docker logs --since 5m -tf pg16
```

## Connect using psql

```bash
❯ docker exec -it pg16 psql -U postgres -c 'create table t()'
CREATE TABLE

❯ docker exec -it pg16 psql -U postgres -c '\d t'
                Table "public.t"
 Column | Type | Collation | Nullable | Default
--------+------+-----------+----------+---------
```

For interactive psql, use:

```bash
docker exec -it pg16 psql -U postgres
```

## Connect any application from outside

To connect an application from the host machine, we need to map ports. For this, we'll destroy this container, and
create a new one, with proper port mapping – noting that `PGDATA` persists (the table we created is there):

```bash
❯ docker stop pg16
pg16

❯ docker rm pg16
pg16

❯ docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  -p 127.0.0.1:15432:5432 \
  postgres:16
8b5370107e1be7d3fd01a3180999a253c53610ca9ab764125b1512f65e83b927

❯ PGPASSWORD=secret psql -hlocalhost -p15432 -U postgres -c '\d t'
Timing is on.
                Table "public.t"
 Column | Type | Collation | Nullable | Default
--------+------+-----------+----------+---------
```

## Custom image with additional extensions

For example, here is how we can create our own image, based on the original one, to include `plpython3u` (continuing to
work with the same `PGDATA`)

```bash
docker stop pg16

docker rm pg16

echo "FROM postgres:16
RUN apt update
RUN apt install -y postgresql-plpython3-16" \
> postgres_plpython3u.Dockerfile

sudo docker build \
  -t postgres-plpython3u:16 \
  -f postgres_plpython3u.Dockerfile \
  .

docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  postgres-plpython3u:16

docker exec -it pg16 \
  psql -U postgres -c 'create extension plpython3u'
```

## Shared memory

If you see an error like this one:

```
> FATAL:  could not resize shared memory segment "/PostgreSQL.12345" to 1048576 bytes: No space left on device1
```

then increase the `--shm-size` value in the `docker run` command.

## How to upgrade Postgres preserving data

1) In-place upgrades:

   - Traditional Docker images for Postgres include binaries only for one major version, so running `pg_upgrade` is not
     possible, unless you extend those images
   - Alternatively, you can use images that include multiple binaries, –
     e.g., [Spilo by Zalando](https://github.com/zalando/spilo).

2) Simple dump/restore (here I show how to downgrade assuming there are no incompatibilities; upgrade can be done in the
   same way):

    ```bash
    docker exec -it pg16 pg_dumpall -U postgres \
      | bzip2 > dumpall.bz2
    
    docker rm -f pg16
    
    rm -rf ~/pgdata
    mkdir ~/pgdata
    
    docker run \
      --detach \
      --name pg15 \
      -e POSTGRES_PASSWORD=secret \
      -v ~/pgdata:/var/lib/postgresql/data \
      --shm-size=128m \
      postgres:15
    
    bzcat dumpall.bz2  \
      | docker exec -i pg15 psql -U postgres \
    >>dump_load.log \
    2> >(tee -a dump_load.err >&2)
    ```
