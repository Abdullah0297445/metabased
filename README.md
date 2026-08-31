# Metabase

Self-hosted [Metabase](https://www.metabase.com) as a docker container, behind a traefik
proxy, with Postgres as its application database.

- traefik terminates TLS. Metabase holds no certificate and publishes no port.
- Postgres holds the application database, rather than the default H2 file.

`metabased` — the **`d`** marks a docker-only deployment: it runs as a container, driven by
docker compose, with nothing installed alongside it.

## What runs here

| Container | Image | Job |
|---|---|---|
| `metabase` | `metabase/metabase` | The whole application: the web UI, the query engine, the scheduler, and the MCP server at `/api/metabase-mcp`. One JVM process. |

It listens on **3000** inside the container and publishes no port. The proxy reaches it on
the network you name in `TRAEFIK_NETWORK`; Postgres answers on `POSTGRES_NETWORK`.

The tag is **exact**, never `latest`. Metabase runs Liquibase migrations at start, a
failure is fatal, and the supported way back is a restore rather than a downgrade — so an
unattended pull would be an unattended, irreversible schema change. See
[Upgrading](#upgrading).

## Before you start

Three things must already exist. None of them is made by this repo.

1. **A traefik proxy runs** on a docker network you can join, and it defines both the
   certificate resolver you name in `RESOLVER_NAME` and the HTTP-to-HTTPS middleware you
   name in `HTTPS_MIDDLEWARE`. traefik issues no certificate for a resolver it does not
   hold.
2. **A Postgres server answers on a docker network you can join**, holding a database and
   a login role for Metabase. **Postgres 14 or newer** — the v0.63 line raised the floor.
   Keep the connection details.
3. **A public DNS A record** for the subdomain points at this machine. A proxy using a
   DNS-01 challenge can issue the certificate before that record exists — but nothing
   answers on the name until it does.

Both networks are joined as **external** — this repo creates neither. Name them in
`TRAEFIK_NETWORK` and `POSTGRES_NETWORK`. `METABASE_DB_HOST` and `METABASE_DB_PORT` are the
address **on that Postgres network**, which for a container is usually its service or
container name and the port it listens on there, not anything routable from your shell. If
your server sits behind a connection pooler, read [Point it at a session-pooled
endpoint](#point-it-at-a-session-pooled-endpoint) before choosing which endpoint to
name.

## Usage

1. `cp .env.example .env`
2. Change every value in `.env`. Generate the encryption key with
   `openssl rand -base64 32`.
3. Confirm `.gitignore` excludes `.env` **before** the first commit.
4. **Back up `.env` to a secure place away from this machine, and back it up again after
   every change to it.** It holds `MB_ENCRYPTION_SECRET_KEY`, which is not in the database
   and cannot be derived. See [The encryption key is a one-way
   door](#the-encryption-key-is-a-one-way-door) — this is the most consequential line in
   the file.
5. `docker compose up -d`
6. `docker compose ps` — the container must reach a healthy state, and it may not show a
   host port binding.
7. `docker logs -f traefik`, on the proxy, to watch the certificate come in. Issuance is
   visible if the proxy logs at `INFO`.
8. Open the subdomain over HTTPS and make the owner account.
9. **Set the Site URL.** Admin → Settings → General → Site URL, to the same `https://`
   address. It is a **database setting, not an environment variable**, which is why
   `MB_SITE_URL` is deliberately absent from the compose file. Leave it wrong and it stays
   wrong: see [Site URL is load-bearing](#site-url-is-load-bearing).

## Configuration

Everything below lives in `.env`, which is gitignored. No real subdomain, host address or
secret belongs in any tracked file.

| Variable | Required | Default | What it is |
|---|---|---|---|
| `METABASE_HOST` | yes | — | The subdomain Metabase answers on. A public DNS A record for it must already point at this machine. traefik reads it in the `Host()` rule of both routers. |
| `TRAEFIK_NETWORK` | yes | — | The existing docker network the proxy is on. Joined as external, and also given to traefik as `traefik.docker.network`. |
| `POSTGRES_NETWORK` | yes | — | The existing docker network the Postgres server answers on. Joined as external. |
| `RESOLVER_NAME` | yes | — | The proxy's certificate resolver. It **must** be the same word the proxy defines. |
| `HTTPS_MIDDLEWARE` | yes | — | The proxy's own HTTP-to-HTTPS middleware, e.g. `redirect-to-https@docker`. Applied to the plain-HTTP router so it is the only thing doing the redirect. |
| `METABASE_DB_HOST` | yes | — | Where Postgres answers **on `POSTGRES_NETWORK`** — usually a container or service name, not a shell-routable address. |
| `METABASE_DB_PORT` | no | `5432` | The port it answers on there. |
| `METABASE_DB_NAME` | yes | — | The database Metabase keeps its own tables in. Its dashboards, questions, users and settings all live here. |
| `METABASE_DB_USER` | yes | — | The login role that owns that database. |
| `METABASE_DB_PASSWORD` | yes | — | That role's password. Postgres keeps no copy you can read back, so keep your own record. |
| `MB_ENCRYPTION_SECRET_KEY` | see below | unset | Encrypts saved credentials at rest. Blank means no encryption. **A one-way door once set.** |
| `MB_AGGREGATED_QUERY_ROW_LIMIT` | no | `10000` | Maximum rows returned for an **aggregated** query. Must stay under 1,048,575. |
| `MB_UNAGGREGATED_QUERY_ROW_LIMIT` | no | `2000` | Maximum rows returned for an **unaggregated** query. Raising it past 2,000 does not change what a table visualization draws. |

The two row limits are **optional knobs**, and the defaults above are Metabase's own. Set
them only if you have a reason — every raised limit is more rows held in the JVM heap and
more rows over the wire, on a container that has one process to lose.

## The encryption key is a one-way door

`MB_ENCRYPTION_SECRET_KEY` encrypts the secret columns of the application database — most
importantly `metabase_database.details`, which holds the **credentials to every data
source you connect**. Without it those credentials sit in clear in the database, and in
every backup taken from it.

**The door only swings one way in practice, and both failure modes are hard stops.** These
five states are observed behaviour on v0.63.15, not inference:

| Application database | Key | What happens |
|---|---|---|
| unencrypted | none | boots — `Saved credentials encryption is DISABLED 🔓` |
| unencrypted | a new key | **boots and encrypts itself**, in place, on that start |
| encrypted | the correct key | boots |
| encrypted | **a wrong key** | **exit 1** — `Database was encrypted with a different key than the MB_ENCRYPTION_SECRET_KEY environment contains` |
| encrypted | **none** | **exit 1** — `Database is encrypted but the MB_ENCRYPTION_SECRET_KEY environment variable was NOT set` |

The last two throw in `check-encryption` **before migrations run**, so Metabase does not
start at all. There is no partial-service degradation to notice and no `enable-encryption`
command to reach for — the family is `remove-encryption` and `rotate-encryption-key`, and
**both of them require the key you have lost.**

So the only exit from a lost key is restoring a dump taken before encryption was turned
on. Past your backup retention there is no exit: you rebuild every dashboard and every
question by hand.

**Keep a copy of the key off this machine** — with your other durable secrets, in whatever
place survives the machine. `.env` is the working copy, not the record.

Turning it on for the first time is one restart: put the key in `.env`, `docker compose up
-d`, and read the log for

```
util.encryption :: Saved credentials encryption is ENABLED for this Metabase instance. 🔐
app-db.setup    :: New MB_ENCRYPTION_SECRET_KEY environment variable set. Encrypting database...
app-db.setup    :: Database encrypted... ✅
```

Do it **on its own**, not in the same window as a version upgrade. One boot doing one
novel thing leaves a failure with one candidate cause, and it keeps the dump you took
beforehand openable without the key.

## Point it at a session-pooled endpoint

If your Postgres sits behind a connection pooler such as PgBouncer, `METABASE_DB_HOST` must
name a **session**-pooled endpoint, not a transaction-pooled one. Metabase holds
connections across statements and keeps state on them, which is exactly what a transaction
pooler takes away. Pointing straight at Postgres with no pooler in between is fine.

The consequence is worth knowing before you tune anything: **on a session-pooled endpoint
each connection Metabase holds pins a server connection for as long as it holds it.**
Metabase's application pool is capped by `MB_APPLICATION_DB_MAX_CONNECTION_POOL_SIZE`,
default **15**, and this deployment runs that default. Raise it and you raise the number of
server connections this one application occupies on a server other applications may share —
so size it against the server's `max_connections`, not against Metabase alone.

Metabase reports its own use in the logs, as `App DB connections: 12/15`. Watch that
before reaching for the knob.

`MB_JDBC_DATA_WAREHOUSE_MAX_CONNECTION_POOL_SIZE`, also **15** by default, is a different
pool: connections out to the data sources you query. It has nothing to do with the
application database or with this endpoint.

## Site URL is load-bearing

`site-url` is a row in the application database, set in Admin, and **not** settable by
environment variable here. Metabase builds more from it than the links in its emails:

- **OAuth discovery for the built-in MCP server is derived from it**, including every
  advertised endpoint and the `WWW-Authenticate` challenge. A client that registers
  against one issuer stops matching when the issuer changes.
- Behind a TLS-terminating proxy it can easily read `http://` while the world reaches you
  on `https://`, because Metabase sees the proxy's plain-HTTP hop.

Set it correctly at install, before any MCP client registers. Changing it later
invalidates those registrations, and re-registering is the only fix.

## Upgrading

An upgrade runs the new version's migrations at start. A failure is fatal, and a
downgrade is **not** the way back: `migrate down` takes a direction rather than a target
and moves one major per run, from the higher binary each time, and it cannot undo work
that happens at start-up rather than in a migration. Metabase's own documentation
recommends restoring a dump. So the tag is exact, and an upgrade never runs by itself.

Every version bump, in this order:

1. **Read every release note between the two versions.** The things that bite are not in
   the migrations: a major can move the sample database's engine, break the driver plugin
   API so third-party JARs need a rebuild, or jump the bundled JVM.
2. **Take a fresh dump of the application database.** Whatever schedule backs that
   database up, "last night" is not "one minute ago". This dump is the rollback.
3. **Rehearse on a copy first, and do it on another machine.** Restore that dump into a
   throwaway Postgres, boot the *new* tag against it, and check the things you would
   notice losing: dashboard, question and user counts against an inventory you took
   beforehand, `/api/health`, and the schema version in the log. The migration itself is
   usually seconds — the rehearsal is what earns the phrase *no data loss*, because the
   dump only proves you can go back.
4. Edit the tag in `docker-compose.yml`.
5. `docker compose up -d`, and watch the migration in the logs.
6. Check the same counts on the real instance, and confirm each data source still
   connects.

**A rehearsal instance must never reach production.** Give it no route to your real
databases — a compose project on no external network is the simplest way, since the data
source's hostname then does not resolve and it fails closed. A restored copy is production
data on whatever machine you put it on; treat it that way, and delete its volumes
afterwards.

Two notes if the application database is encrypted. The rehearsal deliberately does
**not** need the key: everything above still behaves identically, and the one thing that
changes inside the restored copy is that saved credentials cannot be decrypted, so the
data sources fail to connect. That is the expected state, not a fault — do not copy the
key onto the rehearsal machine to make it go away, because that is exactly the exposure
encryption exists to close.

## Health

The container has a healthcheck on `/api/health`, which answers 200 once the application
has started and its database is reachable. `start_period` is 120s, which is the JVM start
plus migrations on a first boot; the check begins failing the container only after that.

## What is backed up, and what is not

**This stack backs nothing up itself.** That is a decision, not an omission.

**The application database** holds the dashboards, the questions, the users, the settings
and the data-source credentials — everything you would mind losing. It lives on a Postgres
server that this stack does not own, so backing it up belongs to whoever runs that server.
Confirm that it is covered, and confirm a restore has been tried.

**`MB_ENCRYPTION_SECRET_KEY` is yours to keep**, and it is in `.env`. A database backup
taken after encryption is turned on restores credentials that nothing can read without it,
and the instance will not start without it either. Back up `.env`, and **keep it apart
from the database backup** — the archive and the key in one place is one loss, not two.

**The `metabase_plugins` volume needs no backup.** Metabase re-downloads its own driver
JARs into it at start. The only thing that would not come back is a third-party driver JAR
you put there by hand — keep your own copy of that, and expect to rebuild it against the
new plugin API after a major upgrade.

## Notes

- No real subdomain, host address, email or secret belongs in a tracked file. Use
  `example.com` and a `${VARIABLE}`, and keep every real value in `.env`.
- `.env` is gitignored. Never commit it.
- `traefik.docker.network` is load-bearing, not belt-and-braces. This container sits on
  two networks, and a proxy with no `--providers.docker.network` default would otherwise
  be free to pick the wrong address.
- HTTP-to-HTTPS redirection is left to the proxy's own middleware, named in
  `HTTPS_MIDDLEWARE`. Metabase's own **Redirect to HTTPS** setting is redundant behind it,
  and turns into a redirect loop if `X-Forwarded-Proto` ever stops arriving.
- `/dev/urandom` is mounted over `/dev/random`. The JVM blocks on a starved entropy pool
  otherwise, which shows up as a start that hangs rather than one that fails.
- The MCP server at `/api/metabase-mcp` is part of the application, on every edition.
  Clients authenticate over OAuth 2.0 against an OAuth server Metabase embeds, and their
  tokens carry the permissions of the account that authorised them — so it is a per-person
  connection, not a shared key. It is governed in Admin, not here.
