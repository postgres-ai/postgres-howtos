Originally from: [tweet](https://twitter.com/samokhvalov/status/1715636738578845831), [LinkedIn post]().

---

# How to quit from psql

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Just enter `quit` (or `exit`). That's it.

Unless you're on Postgres 10 or older – in that case, it's `\q`. Postgres 11 is retiring in a couple of weeks ([the final
minor release, 11.22 is scheduled on November 9](https://postgresql.org/support/versioning/)), so we'll be able to say
that for all supported versions it's just `quit`.

And if you need it to work in non-interactive mode, then it's also `\q`:

- `psql -c '\q'` – this works
- `psql -c 'quit'` – this doesn't

Alternatively, `Ctrl-D` should also work. Being a standard way to exit a console, it works in various shells such as
bash, zsh, irb, python, node, etc.

---

This question remains in the [top-5 on StackOverflow](https://stackoverflow.com/q/9463318/459391) and other places.
