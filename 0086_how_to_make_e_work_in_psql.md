Originally from: [tweet](https://twitter.com/samokhvalov/status/1738267148395688349), [LinkedIn post]().

---

# How to make "\e" work in psql on a new machine ("editor/nano/vi not found")

> I post a new PostgreSQL "howto" article every day. Join me in this
> journey – [subscribe](https://twitter.com/samokhvalov/), provide feedback, share!

Sometimes this happens, when you're attempting to edit a query in psql using the `\e` command:

```
nik=# \e
/usr/bin/sensible-editor: 20: editor: not found
/usr/bin/sensible-editor: 31: nano: not found
/usr/bin/sensible-editor: 20: nano-tiny: not found
/usr/bin/sensible-editor: 20: vi: not found
Couldn't find an editor!
Set the $EDITOR environment variable to your desired editor.
```

Setting the editor is simple (use `nano` or another editor you prefer):

```
\setenv PSQL_EDITOR vim
```

But if you work inside a container, or on a new machine, the desired editor might not yet be installed. You can install
it without leaving `psql`, assuming that there are enough permissions to run installation. For example, inside a 
"standard" Postgres, Debian-based (`sudo` is not needed here):

```
nik=# \! apt update && apt install -y vim
```

👉 And then `\e` starts working!

To make this setting persistent, add this to `~/.bash_profile` (or `~/.zprofile`):

```bash
echo "export PSQL_EDITOR=vim" >> ~/.bash_profile
source ~/.bash_profile
```

For Windows, see 
[this blog post](https://cybertec-postgresql.com/en/psql_editor-fighting-with-sublime-text-under-windows/) 
by [@PavloGolub](https://twitter.com/PavloGolub).

Docs: https://postgresql.org/docs/current/app-psql.html
