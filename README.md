# EMR

The EMR single-sign-on stack: eleven apps brokered by
[Aerem AuthService](https://github.com/8848digital/Aerem_Auth_Service), runnable
with one command.

```bash
git clone --recurse-submodules https://github.com/amir8848Digital/EMR.git
cd EMR
cp .env.example .env     # then fill in the two passwords
docker compose up -d
```

Full setup, troubleshooting and design notes: **[DOCKER.md](DOCKER.md)**.

## What's in here

The apps are **git submodules**, so `--recurse-submodules` matters. This repo
itself holds only the orchestration: the compose file, the images and the docs.

`new--emr/` is the one plain directory in that list — its two apps are
submodules of this repo directly, rather than going through the intermediate
`amir8848Digital/new--emr` repo. Only `new--emr/.env.example` is tracked
alongside them, because `docker-compose.yml` reads `new--emr/.env`; everything
else that repo carried still lives there if you need it.

| Path | App | Port |
|---|---|---|
| `Aerem_Auth_Service` | SSO broker | 4000 |
| `new--emr/emr_node` | new EMR API | 8000 |
| `new--emr/Basecamp_Frontend` | new EMR frontend (Basecamp) | 3001 |
| `CRM/crm-frontend` | CRM | 3000 |
| `PD/Basecamp_Frontend` | Product Development | 3003 |
| `DC/digital-catalog` + `DC/EMR_Catalog_Backend` | Digital catalog | 3002 / 5001 |
| `POS/POS` + `POS/EMR_Catalog_Backend` | POS | 3006 / 5003 |
| `WT/Workers_tab_web` + `DC/EMR_Catalog_Backend` | Workers Tab | 3004 / 5091 |

DC and WT share one backend: `dc_catalog_multidb` carries both, so
`DC/EMR_Catalog_Backend` also answers on WT's port 5091.

Ports are **pinned**: each is registered in AuthService as that client's redirect
and back-channel URL, so renumbering one breaks SSO for that app.

## Two things this repo cannot give you

**The Frappe bench.** `demo-bench` is not tracked here, and it cannot be: none of
its nine apps has an `origin` remote, and `emr_sso` has no remote at all plus
uncommitted work. The copy on disk is the only one in existence — back it up.
Frappe serves CRM and PD, so without it those two have no backend.

**The database.** `demo_site` is ~4.7GB across 612 tables, nearly all real
business data rather than fixtures. It is shared separately by the repo owner;
drop the dump into `docker/db-init/` and it is imported on first start.

## Local secrets

`.env`, every app's `.env`, `.npmrc` and all Frappe `site_config.json` files are
git-ignored because they hold passwords, encryption keys and registry tokens.
Copy `.env.example` and fill it in; never commit the result.
