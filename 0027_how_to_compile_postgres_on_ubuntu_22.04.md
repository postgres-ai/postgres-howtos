Originally from: [tweet](https://twitter.com/samokhvalov/status/1716359165806035147), [LinkedIn post]().

---

# How to compile Postgres on Ubuntu 22.04

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

This post describes how to quickly compile Postgres on Ubuntu 22.04.

The [official docs](https://postgresql.org/docs/current/installation.html). It is very detailed, and it's great to be
used as a reference, but you won't find a concrete list of steps for a particular OS version, e.g., Ubuntu.

This howto is enough to build Postgres from the `master` branch and start using it. Extend this basic set of steps if
needed.

A couple of more notes:

- The current (as of 2023, PG16) docs mention a new approach to building Postgres – [Meson](https://mesonbuild.com/) –
  but it is still considered experimental, we won't use it here.

- The paths for binaries and `PGDATA` that we use here are provided just as examples (suitable for a "quick-n-dirty"
  setup to test, for example, a new patch from the `pgsql-hackers` mailing list).

1. Install required software
2. Get source code
3. Configure
4. Compile and install
5. Create a cluster and start using it

## 1) Install required software

```bash
sudo apt update

sudo apt install -y \
  build-essential libreadline-dev \
  zlib1g-dev flex bison libxml2-dev \
  libxslt-dev libssl-dev libxml2-utils \
  xsltproc ccache
```

## 2) Get source code

```bash
git clone https://gitlab.com/postgres/postgres.git
cd ./postgres
```

## 3) Configure

```bash
mkdir ~/pg16 # consider changing it!

./configure \
  --prefix=$(echo ~/pg16) \
  --with-ssl=openssl \
  --with-python \
  --enable-depend \
  --enable-cassert
```

(Review the list of options.)

## 4) Compile and install

```bash
make -j$(grep -c processor /proc/cpuinfo)
make install
```

## 5) Create a cluster and start using it

```bash
~/pg16/bin/initdb \
  -D ~/pgdata \
  --data-checksums \
  --locale-provider=icu \
  --icu-locale=en

~/pg16/bin/pg_ctl \
  -D ~/pgdata \
  -l pg.log start
```

Now, check:

```bash
$ ~/pg16/bin/psql postgres -c 'show server_version'
 server_version
----------------
 17devel
(1 row)
```
