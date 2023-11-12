Originally from: [tweet](https://twitter.com/samokhvalov/status/1723535079451033697), [LinkedIn post]().

---

# How to install Postgres 16 with plpython3u: Recipes for macOS, Ubuntu, Debian, CentOS, Docker

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

PL/Python is a procedural language extension for PostgreSQL that allows you to write stored procedures and triggers in
Python, a widely-used, high-level, and versatile programming language.

`plpython3u` is the "untrusted" version of PL/Python. This variant allows Python functions to perform operations such as
file I/O, network communication, and other actions that could potentially affect the server's behavior or security.

We used `plpython3u` in
[Day 23: How to use OpenAI APIs right from Postgres to implement semantic search and GPT chat](0023_how_to_use_openai_apis_in_postgres.md),
let's now discuss how to install it.

And something tells me that we'll be using it more in the future, for various tasks.

This howto is for self-managed Postgres only.

## macOS (Homebrew)

```bash
brew tap petere/postgresql
brew install petere/postgresql/postgresql@16

psql postgres \
  -c 'create extension plpython3u'
```

## Ubuntu 22.04 LTS or Debian 12

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

curl -fsSL https://postgresql.org/media/keys/ACCC4CF8.asc \
  | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

sudo apt update
sudo apt install -y \
  postgresql-16 \
  postgresql-contrib-16 \
  postgresql-plpython3-16

sudo -u postgres psql \
  -c 'create extension plpython3u'
```

## CentOS Stream 9

```bash
dnf module reset -y postgresql 
dnf module enable -y postgresql:16

dnf install -y \
  postgresql-server \
  postgresql \
  postgresql-contrib \
  postgresql-plpython3

postgresql-setup --initdb

systemctl enable --now postgresql

sudo -u postgres psql \
  -c 'create extension plpython3u'
```

## Docker

```bash
echo "FROM postgres:16
RUN apt update
RUN apt install -y postgresql-plpython3-16" \
> postgres_plpython3u.Dockerfile

sudo docker build \
  -t postgres-plpython3u:16 \
  -f postgres_plpython3u.Dockerfile \
  .

sudo docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v $(echo ~)/pgdata:/var/lib/postgresql/data \
  postgres-plpython3u:16

sudo docker exec -it pg16 \
  psql -U postgres -c 'create extension plpython3u'
```
