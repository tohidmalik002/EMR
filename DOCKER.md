# Running the whole EMR stack in Docker

One command brings up all eleven apps, the Frappe bench, MariaDB and three Redis
instances:

```bash
docker compose up -d
```

This replaces `./start-all.sh`, which started half the stack as host processes.

---

## What runs

| Service | URL | What it is |
|---|---|---|
| `auth-service` | http://localhost:4000/admin | SSO broker (AuthService) |
| `frappe` | http://demo.localhost:8002 | Frappe desk + REST API, 9 apps |
| `crm` | http://localhost:3000 | CRM frontend |
| `emr-frontend` | http://localhost:3001 | new EMR (Basecamp) |
| `emr-backend` | http://localhost:8000 | new EMR API (`emr_node`) |
| `dc-web` | http://localhost:3002 | Digital catalog |
| `dc-backend` | http://localhost:5001, http://localhost:5091 | Catalog API (DC and WT — one backend on `dc_catalog_multidb`) |
| `pd` | http://localhost:3003 | Product Development frontend |
| `wt-web` | http://localhost:3004 | Workers Tab |
| `pos-web` | http://localhost:3006 | POS |
| `pos-backend` | http://localhost:5003 | Catalog API (POS) |
| `mariadb` | localhost:3308 | `demo_site` — shared by Frappe *and* all three catalog backends |

**Do not renumber these ports casually.** Each one is registered in AuthService
as a client redirect / back-channel URL. Changing a port breaks single sign-on
for that app until you update its client record at
http://localhost:4000/admin.

---

## First-time setup

### 1. Stop anything already on these ports

The host stack and this one want the same ports. Stop the old one first:

```bash
pkill -f "bench start"                                   # Frappe demo bench
cd Aerem_Auth_Service && docker compose down && cd ..     # old AuthService stack
cd new--emr && docker compose -f docker-compose.dev.yml down && cd ..
```

Then stop any `npm run dev` / `node app.js` you started by hand. Careful with
the catalog backends — DC (5001), POS (5003) and WT (5091) are all literally
`node app.js`, so `pkill -f "node app.js"` kills all three at once.

### 2. Configure

```bash
cp .env.example .env
```

Set `MARIADB_ROOT_PASSWORD` to anything, and `FRAPPE_DB_PASSWORD` to the
`db_password` already in `demo-bench/sites/demo.localhost/site_config.json` —
Frappe and the three catalog backends all authenticate with it.

### 3. Load the database

The containers need `demo_site`, and it is **not** in this repo — it is ~4.7GB
across 612 tables, nearly all of it real business data (`Txnd`, `OrdRm`,
`OrdDsg` — millions of rows), not fixtures that could be regenerated. **The repo
owner shares it separately.**

Once you have the dump, drop it in:

```
docker/db-init/10-demo_site.sql.gz
```

Everything in that directory is imported **once**, the first time the `mariadb`
service starts on an empty data volume. See `docker/db-init/README.md`.

### 4. Dependencies

Every app's `node_modules` is **reused from disk, never reinstalled** — see
[Why nothing reinstalls](#why-nothing-reinstalls). If an app has no
`node_modules`, its container runs `npm install` on first start. The one
exception is `dc-web`; install that one on the host first (below).

### 5. Up

```bash
docker compose up -d
docker compose logs -f frappe
```

The **first** `frappe` start builds a Python virtualenv and installs
requirements for all nine bench apps — several minutes. Later starts skip it.

---

## Day to day

```bash
docker compose up -d                  # start everything
docker compose down                   # stop; volumes and data survive
docker compose logs -f crm            # follow one app
docker compose restart dc-backend     # reload a backend after an .env change
docker compose exec frappe bench --site demo.localhost console
docker compose exec frappe bench --site demo.localhost migrate
docker compose exec mariadb mariadb -u root -p demo_site
```

Source is bind-mounted, so edits reload live: nodemon for the backends, Next's
dev server for the frontends, `bench watch` for Frappe assets. No rebuild needed
unless you change a `Dockerfile`.

---

## Why nothing reinstalls

The containers deliberately reuse each app's existing `node_modules` instead of
installing their own. This is a correctness requirement, not an optimisation:

- **`DC/digital-catalog` cannot be installed from scratch here.** `apps/web`
  depends on `@8848digital/catalyst` and `@8848digital/offline-kit`, published to
  GitHub Packages. The repo's `.npmrc` routes that scope to
  `npm.pkg.github.com` and expects a read token in `~/.npmrc`, which does not
  exist on this machine. Any install that re-resolves the graph dies with
  `ERR_PNPM_FETCH_401`. The packages are only present because they are already
  unpacked under `node_modules/.pnpm/`.
- So `dc-web` runs `next` **directly**, not through `pnpm`. pnpm v11 verifies the
  dependency tree before running a script, and here it would want to purge and
  rebuild `node_modules` (the on-disk tree is `isolated`; `.npmrc` asks for
  `hoisted`). That purge is unrecoverable without a token.

To make `dc-web` reproducible, put a GitHub Packages read token in `~/.npmrc`:

```
//npm.pkg.github.com/:_authToken=YOUR_GITHUB_READ_TOKEN
```

The images are Debian, not Alpine, for a related reason: the bind-mounted
`node_modules` trees contain prebuilt glibc binaries (`next-swc`, `sharp`,
`rollup`, `better-sqlite3`) that musl cannot load. `node:20` matches the host's
Node 20.20.2 so node-ABI modules load unchanged; `dc-web` gets `node:22` because
pnpm@11 refuses to run on Node 20.

---

## How the Frappe bench works here

The bench is bind-mounted rather than built into the image, because **it cannot
be cloned**: none of the nine apps has an `origin` remote, several sit on local
feature branches, and `emr_sso` has no remote at all plus uncommitted work. The
copy in `demo-bench/apps` is the only one that exists. Back it up accordingly.

Two files are layered over the mount so the **host** bench keeps working
unchanged:

| Container file | Why |
|---|---|
| `docker/frappe/common_site_config.json` | points at the `mariadb` and `frappe-redis-*` services instead of `127.0.0.1` |
| `docker/frappe/Procfile` | drops the `redis_cache`/`redis_queue` lines (Redis is its own service here, and the host configs hardcode host absolute paths) and the hardcoded nvm `PATH` |

`env/` is a named volume, **not** shared with the host: the host's virtualenv has
absolute host paths and a host-built interpreter in it. Both benches get their own
interpreter over the same source.

Python 3.14 is not a choice — `apps/frappe/pyproject.toml` pins
`requires-python = ">=3.14,<3.15"`.

### After adding a bench app or changing a pyproject

```bash
docker compose exec frappe rm env/.requirements-installed
docker compose restart frappe
```

---

## Databases

- **`mariadb` service** holds `demo_site`, shared by Frappe and all three
  catalog backends. It is published on **3308**, because 3306 (host MariaDB) and
  3307 (`mariadb-emr6122`) are usually taken.
- **It is a copy.** Your host MariaDB on :3306 is never touched by this stack.
- **`emr-backend` does not use it.** `emr_node` talks to an external MSSQL
  server, per `DB_HOST` in `new--emr/.env`. Nothing was containerised for that.

Re-import the dump (destroys only the container's copy, then reloads
`docker/db-init/` on the next start):

```bash
docker compose down
docker volume rm emr-stack_mariadb-data
docker compose up -d mariadb
```

---

## How app config is layered

Each app keeps reading **its own** `.env` exactly as before. The root `.env`
configures only compose. On top of that, `docker-compose.yml` overrides the few
values that must differ inside a container:

- `DB_SERVER` → `mariadb` (was `127.0.0.1`)
- `AUTH_SERVICE_URL` → `http://auth-service:4000` (container to container)
- `AUTH_SERVICE_PUBLIC_URL` → stays `http://localhost:4000` (the browser has to
  reach it from the host)
- `NEXT_PUBLIC_*` are left alone — they are read by the browser, not the server

`auth-service` gets `host.localhost`, `demo.localhost` and `dc.localhost` mapped
to the host gateway, so the back-channel logout URLs already stored in its
`clients.db` keep resolving. Nothing needs re-registering.

---

## What has been verified

Checked on this machine, with the `mariadb` service up and an **unseeded**
database:

| Check | Result |
|---|---|
| `mariadb` service | healthy; `utf8mb4` / `utf8mb4_unicode_ci`; `demo_site` user authenticates |
| Node images | `node -v` = v20.20.2, identical to the host, so native modules load unchanged |
| Bind-mounted `node_modules` | reused, not reinstalled; `next` runs from each app's own tree |
| `pd` | dev server returns **HTTP 200**, binds `0.0.0.0`, reads its `.env` |
| `dc-web` | dev server returns **HTTP 307** (its middleware redirect to `/login`) — `@8848digital/catalyst` resolves and **no pnpm purge occurred** |
| `dc-backend` | resolves `mariadb`, authenticates, boots and listens on 5001 |
| `frappe` image | Python **3.14.8** (same as host), bench 5.31.0, wkhtmltopdf **0.12.6.1 with patched qt** |
| `frappe` container | runs as uid 1000 (`frappe`), not root, so bench's writes into `apps/`+`sites/` stay yours |
| `frappe` overlays | `db_host`=`mariadb`, `redis_cache`=`redis://frappe-redis-cache:6379`; Procfile has 5 process lines, no redis |
| `frappe` bench mount | all 9 apps and `demo.localhost` visible; `env/` volume writable |
| `frappe` venv + requirements | built from scratch in the container; **all 9 bench apps' python deps installed** under Python 3.14 |
| `frappe` bench CLI | `bench --site demo.localhost version` lists all 9 apps with branches, incl. the disk-only `emr_sso` |
| `pos-backend` | reaches `mariadb`, authenticates, boots on 5003 |
| All 12 images (11 apps + `frappe`) | build clean |

Not yet exercised end-to-end, because it needs the ~4.7GB database loaded and the
host stack stopped: a full `docker compose up -d` with the Frappe bench actually
serving pages, and an SSO login crossing apps. Every backend above stops exactly
at missing *data* (`Table 'demo_site.Param' doesn't exist`), which is the
database-load step, not the wiring.

The first `frappe` start builds its virtualenv and installs nine apps' worth of
Python dependencies. That took roughly ten minutes here; later starts skip it via
a marker file at `env/.requirements-installed`.

---

## Troubleshooting

**`port is already allocated`** — the host stack is still running. See step 1.

**`Unable to acquire lock at /app/.next/dev/lock`** — a Next 16 app (currently
CRM) is already running `next dev` against the same source tree from the host.
`.next/` lives in the bind mount, so the host and container dev servers cannot
share it; the lock is what stops two servers corrupting one build directory.
Stop the host one. (Next 15 apps — PD, DC, POS, WT — do not take this lock, so
they start regardless, but running both is still a bad idea.)

**Frappe 404s every page** — Frappe serves per `Host` header. Use
`http://demo.localhost:8002`, not `http://localhost:8002`.

**A backend ignores an `.env` edit** — nodemon does not watch `.env`.
`docker compose restart <service>`.

**`dc-web` exits complaining about `@8848digital`** — its `node_modules` is
missing or was purged. Reinstall on the host with a token in `~/.npmrc`, or
restore the tree from a backup.

**Frappe can't reach the database** — check the dump actually applied:
`docker compose exec mariadb mariadb -u root -p -e "SHOW DATABASES"`. An init
script only runs on a *fresh* volume.

**A tab stays logged in after signing out elsewhere** — expected. Back-channel
logout revokes server-side only; nothing is pushed to an open browser tab. It
clears on the next API call that returns 401 `SESSION_REVOKED`.
