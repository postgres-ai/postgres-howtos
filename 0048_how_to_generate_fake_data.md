Originally from: [tweet](https://twitter.com/samokhvalov/status/1723973291692691581), [LinkedIn post]().

---

# How to generate fake data

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

## Simple numbers

Sequential numbers from 1 to 5:

```sql
nik=# select i from generate_series(1, 5) as i;
 i
---
 1
 2
 3
 4
 5
(5 rows)
```

5 random `BIGINT` numbers from 0 to 100:

```sql
nik=# select (random() * 100)::int8 from generate_series(1, 5);
 int8
------
   85
   61
   44
   70
   16
(5 rows)
```

Note that per the [docs](https://postgresql.org/docs/current/functions-math.html#FUNCTIONS-MATH-RANDOM-TABLE), random() 

> uses a deterministic pseudo-random number generator. It is fast but not suitable for cryptographic applications...
 
We shouldn't use it for tasks as token or password generation (for that, use the library called `pgcrypto`). 
But it is okay to use it for pure random data generation (not for obfuscation).

Starting with Postgres 16, there is also 
[random_normal(mean, stddev)](https://postgresql.org/docs/16/functions-math.html#FUNCTIONS-MATH-RANDOM-TABLE):

> Returns a random value from the normal distribution with the given parameters; `mean` defaults to 0.0 and `stddev`
> defaults to 1.0

Generating 1 million numbers and checking their distribution:

```sql
nik=# with data (r) as (
  select random_normal()
  from generate_series(1, 1000000)
)
select
  width_bucket(r, -3, 3, 5) as bucket,
  count(*) as frequency
from data
group by bucket
order by bucket;

 bucket | frequency
--------+-----------
      0 |      1374
      1 |     34370
      2 |    238786
      3 |    450859
      4 |    238601
      5 |     34627
      6 |      1383
(7 rows)
```

Again, neither `random()` nor `random_normal()` should be used as cryptographically strong random number generators –
for that, use `pgcrypto`. Otherwise, knowing one value, one could "guess" the next one:

```sql
nik=# set seed to 0.1234;
SET

nik=# select random_normal();
    random_normal
----------------------
 -0.32020779450641174
(1 row)

nik=# select random_normal();
   random_normal
--------------------
 0.8995226247227294
(1 row)

nik=# set seed to 0.1234; -- start again
SET

nik=# select random_normal();
    random_normal
----------------------
 -0.32020779450641174
(1 row)

nik=# select random_normal();
   random_normal
--------------------
 0.8995226247227294
(1 row)
```

## Timestamps, dates, intervals

Timestamps (with timezone) for January 2024, starting with 2024-01-01, with 7-day shift:

```sql
nik=# show timezone;
      TimeZone
---------------------
 America/Los_Angeles
(1 row)

nik=# select i from generate_series(timestamptz '2024-01-01', timestamptz '2024-01-31', interval '7 day') i;
           i
------------------------
 2024-01-01 00:00:00-08
 2024-01-08 00:00:00-08
 2024-01-15 00:00:00-08
 2024-01-22 00:00:00-08
 2024-01-29 00:00:00-08
(5 rows)
```

Generate 3 random timestamps for the previous week (useful for filling columns such as `created_at`):

```sql
nik=# select
  date_trunc('week', now())
  - interval '7 day'
  + format('%s day', (random() * 7))::interval
from generate_series(1, 3);

           ?column?
-------------------------------
 2023-10-31 00:50:59.503352-07
 2023-11-03 11:25:39.770384-07
 2023-11-03 13:43:27.087973-07
(3 rows)
```

Generate a random birthdate for a person aged 18-100 years:

```sql
  nik=# select
  (
    now()
    - format('%s day', 365 * 18)::interval
    - format('%s day', 365 * random() * 82)::interval
  )::date;
    date

------------
 1954-01-17
(1 row)
```

## Pseudowords

Generate a pseudoword consisting of 2-12 lowercase Latin letters:

```sql
nik=# select string_agg(chr((random() * 25)::int + 97), '')
from generate_series(1, 2 + (10 * random())::int);
 string_agg
------------
 yegwrsl
(1 row)

nik=# select string_agg(chr((random() * 25)::int + 97), '')
from generate_series(1, 2 + (10 * random())::int);
 string_agg
------------
 wusapjx
(1 row)
```

Generate a "sentence" consisting of 5-10 such words:

```sql
nik=# select string_agg(w, ' ')
from
  generate_series(1, 5) as i,
  lateral (
    select string_agg(chr((random() * 25)::int + 97), ''), i
    from generate_series(1, 2 + (10 * random())::int + i - i)
  ) as words(w);
             string_agg
-------------------------------------
 uvo bwp kcypvcnctui tn demkfnxruwxk
(1 row)
```

Note `LATERAL` and the trick with "idle" references to the outer generator (`, i` and `+i - i`) to generate new random
values in each iteration.

## Normal words, names, emails, SSNs, etc. (Faker)

[Faker](https://faker.readthedocs.io/en/master/) is a Python library that enables you to generate fake data such as
names, addresses, phone numbers, and more. For other languages:

- [Faker for Ruby](https://github.com/faker-ruby/faker)
- [Faker for Go](https://github.com/go-faker/faker)
- [Faker for Java](https://github.com/DiUS/java-faker)
- [Faker for Rust](https://github.com/cksac/fake-rs)
- [Faker for JavaScript](https://github.com/faker-js/faker)

There are several options to use Faker for Python:

- A regular Python program with Postgres connection (boring; but would work with any Postgres including RDS).
- [postgresql_faker](https://gitlab.com/dalibo/postgresql_faker/)
- PL/Python functions.

Here, we'll demonstrate the use of the latter approach, with the "untrusted" version of PL/Python,
([Day 47: How to install Postgres 16 with plpython3u](0047_how_to_install_postgres_16_with_plpython3u.md); N/A for
managed Postgres services such as RDS; note that in this case, the "trusted" version should suit too).

```sql
nik=# create or replace function generate_random_sentence(
    min_length int,
    max_length int
  ) returns text
  as $$
    from faker import Faker
    import random

    if min_length > max_length:
      raise ValueError('min_length > max_length')

    fake = Faker()

    sentence_length = random.randint(min_length, max_length)

    return ' '.join(fake.words(nb=sentence_length))
  $$ language plpython3u;
CREATE FUNCTION

nik=# select generate_random_sentence(7, 15);
                            generate_random_sentence
---------------------------------------------------------------------------------
 operation day down forward foreign left your anything clear age seven memory as
(1 row)
```

A function to generate names, emails, and SSNs:

```sql
nik=# create or replace function generate_faker_data(
    data_type text,
    locale text default 'en_US'
)
returns text as $$
  from faker import Faker

  fake = Faker(locale)

  if data_type == 'email':
    result = http://fake.email()
  elif data_type == 'lastname':
    result = fake.last_name()
  elif data_type == 'firstname':
    result = fake.first_name()
  elif data_type == 'ssn':
    result = fake.ssn()
  else:
    raise Exception('Invalid type')

  return result
$$ language plpython3u;

select
  generate_faker_data('firstname', locale) as firstname,
  generate_faker_data('lastname', locale) as lastname,
  generate_faker_data('ssn') as "SSN";
CREATE FUNCTION

nik=# select
  locale,
  generate_faker_data('firstname', locale) as firstname,
  generate_faker_data('lastname', locale) as lastname
from
  (values ('en_US'), ('uk_UA'), ('it_IT')) as _(locale);
 locale | firstname | lastname
--------+-----------+-----------
 en_US  | Ashley    | Rodgers
 uk_UA  | Анастасія | Матвієнко
 it_IT  | Carolina  | Donatoni
(3 rows)

nik=# select generate_faker_data('ssn');
 generate_faker_data
---------------------
 008-47-2950
(1 row)

nik=# select generate_faker_data('email', 'es') from generate_series(1, 5);
   generate_faker_data
-------------------------
 isidoro42@example.net
 anselma04@example.com
 torreatilio@example.com
 natanael39@example.org
 teodosio79@example.net
(5 rows)
```
