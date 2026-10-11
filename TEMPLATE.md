# Twenty v2 CRM Sales Workspace — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Twenty v2 CRM Sales Workspace** (version label `v2` = Twenty major version; Twenty tags releases `v2.x.y`) |
| Description | Modern open-source CRM with background workers and S3 storage. Own your data. (77 chars, unchanged from LISTINGS.md) |
| Category | Other |
| Tags | twenty, crm, sales, open-source, salesforce-alternative, postgres |
| User input at deploy | **None.** All secrets are generated. Optional later: SMTP, Google / Microsoft OAuth (mail + calendar sync). |

## Base market template

- **`nAL3hA` "Twenty CRM"** (1,854 projects, 660 active) — chosen as base. It is already on v2 with a Railway Bucket and a separate worker, and it works. Kept: the service split (server + worker), the Bucket wiring, `?family=0` on the Redis URL, `/healthz`.
  Fixed: `latest` tag, plain `postgres:16` with a broken `PGHOST=db` variable, no `ENCRYPTION_KEY`, a volume on the server at a path Twenty does not use, migrations at container start, no mail / OAuth variables.
- `twenty-open-source-crm` (98 active) — MinIO (excluded by the standard), `latest`. Nothing taken.
- `twenty-or-the-1-open-source-crm` (74 active) — Bucket, but `latest`, volumes on both server and worker, healthcheck `/`. Nothing taken.
- No `_upstream` repos for this app. Market templates carry no license → reference only, no text copied.

**Honest assessment vs. the leader:** the leader is functionally close to this template. The gain is hardening and readiness, not a different architecture: pinned version, current encryption setup, migrations that are safe with replicas, a stateless (scalable) server, Railway's standard Postgres, and e-mail / calendar sync that can be switched on without editing the service graph. Expect the same idle cost; the main user-visible difference is that Gmail/Outlook sync and SMTP work by filling variables (or the Admin Panel) instead of being absent.

## Pinned versions

| Service | Image | Tag | Release date | Checked |
|---|---|---|---|---|
| Twenty, Twenty Worker | `twentycrm/twenty` | `v2.45.8` | 2026-10-08 (patch of release `twenty/v2.45.0`, 2026-10-05) | 2026-10-10 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl` | `18` | Railway base | 2026-10-10 |
| Redis | `redis` | `8.2` | Railway base | 2026-10-10 |

`v2.46.0` was pushed to Docker Hub on 2026-10-09 without a GitHub release yet; this template stays on the newest patch of the last published release line. Upgrade path: change the tag on **both** Twenty and Twenty Worker and redeploy. Since v1.23 Twenty supports cross-version upgrades; the pre-deploy command runs `database:init` (first boot) or `upgrade` (later boots). Back up Postgres first (Railway Postgres backups or `pg_dump`). Twenty v2.34+ needs Postgres ≥ 15 (satisfied).

## Architecture

```
Internet ──HTTPS──▶ Twenty (NestJS API + web app, :3000, public domain, stateless)
                      │  pre-deploy: wait for Postgres → database:init / upgrade → cache flush
                      ├─▶ Postgres  (core + workspace schemas)
                      ├─▶ Redis     (cache + BullMQ queues, noeviction)
                      └─▶ Bucket    (attachments, avatars, files; S3 API)
Twenty Worker (yarn worker:prod) ─▶ Postgres, Redis, Bucket   (mail/calendar sync, workflows, imports, webhooks)
```

### Deviations from upstream compose (and why)

- Upstream mounts a `server-local-data` volume on server and worker (local storage). Railway volumes cannot be shared, so both use the **Bucket** (`STORAGE_TYPE=S_3`); the server therefore has no volume and can run several replicas.
- Upstream runs migrations in the server's entrypoint. Here the server runs with `DISABLE_DB_MIGRATIONS=true` and the same entrypoint is executed once as the **pre-deploy command** (with `DISABLE_DB_MIGRATIONS=false`), after waiting for Postgres. Replicas never migrate concurrently and a failed migration stops the deploy instead of crash-looping the server.
- Postgres is Railway's `postgres-ssl:18` instead of `postgres:16` (Twenty requires ≥ 15).
- Redis keeps upstream's `--maxmemory-policy noeviction` (required by BullMQ) on top of Railway's base Redis.
- Mail / OAuth variables are pre-declared (upstream ships them commented out). Twenty also stores these in the database via **Settings → Admin Panel → Config Variables**; a value set there overrides the Railway variable.

## Services

All services are created together from the template; references resolve at deploy time. Values are entered exactly as shown (Railway resolves `${{...}}`). Empty values are optional and may be left blank.

### Twenty

- **Source:** Docker image `twentycrm/twenty:v2.45.8`
- **Start command:** image default
- **Pre-deploy command:** `/bin/sh -c "i=0; until psql $PG_DATABASE_URL -tAc 'select 1' >/dev/null 2>&1; do i=$((i+1)); if [ $i -ge 90 ]; then echo 'Postgres not reachable'; exit 1; fi; sleep 2; done; DISABLE_DB_MIGRATIONS=false DISABLE_CRON_JOBS_REGISTRATION=true exec /app/entrypoint.sh true"`
- **Healthcheck:** `/healthz` (timeout 300 s)
- **Public domain:** yes → target port `3000`
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port Railway routes to and healthchecks (must equal NODE_PORT). |
| `NODE_PORT` | `3000` | Port the Twenty server listens on. |
| `SERVER_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Public URL of this instance (links, OAuth callbacks). Update it when you add a custom domain. |
| `PG_DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres connection string (private network). |
| `REDIS_URL` | `${{Redis.REDIS_URL}}?family=0` | Redis for cache and BullMQ queues. family=0 lets ioredis use IPv4 or IPv6 on the private network. |
| `ENCRYPTION_KEY` | `${{secret(48)}}` | Primary key for at-rest encryption of stored secrets (connected-account tokens, API keys). Never change it without a rotation (see FALLBACK_ENCRYPTION_KEY in Twenty docs). |
| `APP_SECRET` | `${{secret(48)}}` | Legacy application secret, kept for compatibility and as fallback key. Do not change after first boot. |
| `STORAGE_TYPE` | `S_3` | File storage driver. S_3 keeps attachments in the Railway Bucket so server and worker share them and the server stays stateless. |
| `STORAGE_S3_NAME` | `${{Bucket.BUCKET}}` | Bucket name (Railway Bucket). |
| `STORAGE_S3_REGION` | `${{Bucket.REGION}}` | Bucket region (Railway Bucket). |
| `STORAGE_S3_ENDPOINT` | `${{Bucket.ENDPOINT}}` | S3 endpoint (Railway Bucket). |
| `STORAGE_S3_ACCESS_KEY_ID` | `${{Bucket.ACCESS_KEY_ID}}` | Bucket access key. |
| `STORAGE_S3_SECRET_ACCESS_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Bucket secret key. |
| `LOGIC_FUNCTION_TYPE` | `DISABLED` | Logic functions / workflow code steps. LOCAL runs user code unsandboxed inside the container: enable only if every workspace member is trusted. |
| `EMAIL_DRIVER` | `LOGGER` | LOGGER writes e-mails to the logs. Set SMTP (and the EMAIL_SMTP_* values) to send invitations and password resets. Can also be set in Settings > Admin Panel. |
| `EMAIL_FROM_ADDRESS` | *(empty)* | Sender address for system e-mails, e.g. crm@yourcompany.com. Optional. |
| `EMAIL_FROM_NAME` | `Twenty CRM` | Sender display name for system e-mails. |
| `EMAIL_SMTP_HOST` | *(empty)* | SMTP host, e.g. smtp.gmail.com or smtp.office365.com. Optional. |
| `EMAIL_SMTP_PORT` | `587` | SMTP port (587 STARTTLS, 465 TLS). |
| `EMAIL_SMTP_USER` | *(empty)* | SMTP username. Optional. |
| `EMAIL_SMTP_PASSWORD` | *(empty)* | SMTP password or app password. Optional. |
| `AUTH_GOOGLE_ENABLED` | `false` | Allow 'Sign in with Google' (needs the Google client below). |
| `MESSAGING_PROVIDER_GMAIL_ENABLED` | `false` | Enable Gmail sync (needs the Google client below). |
| `CALENDAR_PROVIDER_GOOGLE_ENABLED` | `false` | Enable Google Calendar sync (needs the Google client below). |
| `AUTH_GOOGLE_CLIENT_ID` | *(empty)* | Google OAuth client ID (Google Cloud console). Optional. |
| `AUTH_GOOGLE_CLIENT_SECRET` | *(empty)* | Google OAuth client secret. Optional. |
| `AUTH_GOOGLE_CALLBACK_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}/auth/google/redirect` | Authorised redirect URI for Google sign-in. Register it in Google Cloud. |
| `AUTH_GOOGLE_APIS_CALLBACK_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}/auth/google-apis/get-access-token` | Authorised redirect URI for Gmail/Calendar sync. Register it in Google Cloud. |
| `AUTH_MICROSOFT_ENABLED` | `false` | Allow 'Sign in with Microsoft' (needs the Microsoft app below). |
| `MESSAGING_PROVIDER_MICROSOFT_ENABLED` | `false` | Enable Outlook / Microsoft 365 mail sync. |
| `CALENDAR_PROVIDER_MICROSOFT_ENABLED` | `false` | Enable Microsoft 365 calendar sync. |
| `AUTH_MICROSOFT_CLIENT_ID` | *(empty)* | Microsoft Entra app (client) ID. Optional. |
| `AUTH_MICROSOFT_CLIENT_SECRET` | *(empty)* | Microsoft Entra client secret. Optional. |
| `AUTH_MICROSOFT_CALLBACK_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}/auth/microsoft/redirect` | Redirect URI for Microsoft sign-in. Register it in Entra ID. |
| `AUTH_MICROSOFT_APIS_CALLBACK_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}/auth/microsoft-apis/get-access-token` | Redirect URI for Microsoft mail/calendar sync. Register it in Entra ID. |
| `DISABLE_DB_MIGRATIONS` | `true` | Migrations run in the pre-deploy command, not at container start, so replicas never migrate concurrently. |
| `DISABLE_CRON_JOBS_REGISTRATION` | `false` | The server registers Twenty's background cron jobs on start (idempotent). |

### Twenty Worker

- **Source:** Docker image `twentycrm/twenty:v2.45.8`
- **Start command:** `yarn worker:prod`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `SERVER_URL` | `${{Twenty.SERVER_URL}}` | Shared with the server: references Twenty.SERVER_URL (edit it there). |
| `PG_DATABASE_URL` | `${{Twenty.PG_DATABASE_URL}}` | Shared with the server: references Twenty.PG_DATABASE_URL (edit it there). |
| `REDIS_URL` | `${{Twenty.REDIS_URL}}` | Shared with the server: references Twenty.REDIS_URL (edit it there). |
| `ENCRYPTION_KEY` | `${{Twenty.ENCRYPTION_KEY}}` | Shared with the server: references Twenty.ENCRYPTION_KEY (edit it there). |
| `APP_SECRET` | `${{Twenty.APP_SECRET}}` | Shared with the server: references Twenty.APP_SECRET (edit it there). |
| `STORAGE_TYPE` | `${{Twenty.STORAGE_TYPE}}` | Shared with the server: references Twenty.STORAGE_TYPE (edit it there). |
| `STORAGE_S3_NAME` | `${{Twenty.STORAGE_S3_NAME}}` | Shared with the server: references Twenty.STORAGE_S3_NAME (edit it there). |
| `STORAGE_S3_REGION` | `${{Twenty.STORAGE_S3_REGION}}` | Shared with the server: references Twenty.STORAGE_S3_REGION (edit it there). |
| `STORAGE_S3_ENDPOINT` | `${{Twenty.STORAGE_S3_ENDPOINT}}` | Shared with the server: references Twenty.STORAGE_S3_ENDPOINT (edit it there). |
| `STORAGE_S3_ACCESS_KEY_ID` | `${{Twenty.STORAGE_S3_ACCESS_KEY_ID}}` | Shared with the server: references Twenty.STORAGE_S3_ACCESS_KEY_ID (edit it there). |
| `STORAGE_S3_SECRET_ACCESS_KEY` | `${{Twenty.STORAGE_S3_SECRET_ACCESS_KEY}}` | Shared with the server: references Twenty.STORAGE_S3_SECRET_ACCESS_KEY (edit it there). |
| `LOGIC_FUNCTION_TYPE` | `${{Twenty.LOGIC_FUNCTION_TYPE}}` | Shared with the server: references Twenty.LOGIC_FUNCTION_TYPE (edit it there). |
| `EMAIL_DRIVER` | `${{Twenty.EMAIL_DRIVER}}` | Shared with the server: references Twenty.EMAIL_DRIVER (edit it there). |
| `EMAIL_FROM_ADDRESS` | `${{Twenty.EMAIL_FROM_ADDRESS}}` | Shared with the server: references Twenty.EMAIL_FROM_ADDRESS (edit it there). |
| `EMAIL_FROM_NAME` | `${{Twenty.EMAIL_FROM_NAME}}` | Shared with the server: references Twenty.EMAIL_FROM_NAME (edit it there). |
| `EMAIL_SMTP_HOST` | `${{Twenty.EMAIL_SMTP_HOST}}` | Shared with the server: references Twenty.EMAIL_SMTP_HOST (edit it there). |
| `EMAIL_SMTP_PORT` | `${{Twenty.EMAIL_SMTP_PORT}}` | Shared with the server: references Twenty.EMAIL_SMTP_PORT (edit it there). |
| `EMAIL_SMTP_USER` | `${{Twenty.EMAIL_SMTP_USER}}` | Shared with the server: references Twenty.EMAIL_SMTP_USER (edit it there). |
| `EMAIL_SMTP_PASSWORD` | `${{Twenty.EMAIL_SMTP_PASSWORD}}` | Shared with the server: references Twenty.EMAIL_SMTP_PASSWORD (edit it there). |
| `AUTH_GOOGLE_ENABLED` | `${{Twenty.AUTH_GOOGLE_ENABLED}}` | Shared with the server: references Twenty.AUTH_GOOGLE_ENABLED (edit it there). |
| `MESSAGING_PROVIDER_GMAIL_ENABLED` | `${{Twenty.MESSAGING_PROVIDER_GMAIL_ENABLED}}` | Shared with the server: references Twenty.MESSAGING_PROVIDER_GMAIL_ENABLED (edit it there). |
| `CALENDAR_PROVIDER_GOOGLE_ENABLED` | `${{Twenty.CALENDAR_PROVIDER_GOOGLE_ENABLED}}` | Shared with the server: references Twenty.CALENDAR_PROVIDER_GOOGLE_ENABLED (edit it there). |
| `AUTH_GOOGLE_CLIENT_ID` | `${{Twenty.AUTH_GOOGLE_CLIENT_ID}}` | Shared with the server: references Twenty.AUTH_GOOGLE_CLIENT_ID (edit it there). |
| `AUTH_GOOGLE_CLIENT_SECRET` | `${{Twenty.AUTH_GOOGLE_CLIENT_SECRET}}` | Shared with the server: references Twenty.AUTH_GOOGLE_CLIENT_SECRET (edit it there). |
| `AUTH_GOOGLE_CALLBACK_URL` | `${{Twenty.AUTH_GOOGLE_CALLBACK_URL}}` | Shared with the server: references Twenty.AUTH_GOOGLE_CALLBACK_URL (edit it there). |
| `AUTH_GOOGLE_APIS_CALLBACK_URL` | `${{Twenty.AUTH_GOOGLE_APIS_CALLBACK_URL}}` | Shared with the server: references Twenty.AUTH_GOOGLE_APIS_CALLBACK_URL (edit it there). |
| `AUTH_MICROSOFT_ENABLED` | `${{Twenty.AUTH_MICROSOFT_ENABLED}}` | Shared with the server: references Twenty.AUTH_MICROSOFT_ENABLED (edit it there). |
| `MESSAGING_PROVIDER_MICROSOFT_ENABLED` | `${{Twenty.MESSAGING_PROVIDER_MICROSOFT_ENABLED}}` | Shared with the server: references Twenty.MESSAGING_PROVIDER_MICROSOFT_ENABLED (edit it there). |
| `CALENDAR_PROVIDER_MICROSOFT_ENABLED` | `${{Twenty.CALENDAR_PROVIDER_MICROSOFT_ENABLED}}` | Shared with the server: references Twenty.CALENDAR_PROVIDER_MICROSOFT_ENABLED (edit it there). |
| `AUTH_MICROSOFT_CLIENT_ID` | `${{Twenty.AUTH_MICROSOFT_CLIENT_ID}}` | Shared with the server: references Twenty.AUTH_MICROSOFT_CLIENT_ID (edit it there). |
| `AUTH_MICROSOFT_CLIENT_SECRET` | `${{Twenty.AUTH_MICROSOFT_CLIENT_SECRET}}` | Shared with the server: references Twenty.AUTH_MICROSOFT_CLIENT_SECRET (edit it there). |
| `AUTH_MICROSOFT_CALLBACK_URL` | `${{Twenty.AUTH_MICROSOFT_CALLBACK_URL}}` | Shared with the server: references Twenty.AUTH_MICROSOFT_CALLBACK_URL (edit it there). |
| `AUTH_MICROSOFT_APIS_CALLBACK_URL` | `${{Twenty.AUTH_MICROSOFT_APIS_CALLBACK_URL}}` | Shared with the server: references Twenty.AUTH_MICROSOFT_APIS_CALLBACK_URL (edit it there). |
| `DISABLE_DB_MIGRATIONS` | `true` | The worker never runs migrations. |
| `DISABLE_CRON_JOBS_REGISTRATION` | `true` | Cron jobs are registered by the server. |

### Postgres

- **Source:** Docker image `ghcr.io/railwayapp-templates/postgres-ssl:18`
- **Start command:** image default
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/postgresql/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Location where the database will be initialized. |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `PGPORT` | `5432` | Port to connect to Postgres. |
| `PGUSER` | `${{ POSTGRES_USER }}` | Required variable for Data panel. |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Required variable for the data panel. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Required variable for Data panel. |
| `POSTGRES_DB` | `railway` | Default database created when image is started. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | URL to connect to Postgres database. |
| `POSTGRES_USER` | `postgres` | User to connect to Postgres DB. |
| `SSL_CERT_DAYS` | `820` | SSL certificate expiry in days. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Password to connect to DB. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Allow Postgres to cleanly shut down. |

### Redis

- **Source:** Docker image `redis:8.2`
- **Start command:** `/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH --maxmemory-policy noeviction"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private network hostname of the Redis service, only resolvable from services in the same environment. |
| `REDISPORT` | `6379` | Port that Redis listens on. |
| `REDISUSER` | `default` | Username for authenticating with Redis. |
| `REDIS_URL` | `redis://${{ REDISUSER }}:${{ REDIS_PASSWORD }}@${{ REDISHOST }}:${{ REDISPORT }}` | Connection string for connecting to Redis using the private network. |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | Alias of REDIS_PASSWORD for clients that expect the unseparated name. |
| `REDIS_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Randomly generated password for authenticating with Redis. |

### Bucket (Railway Bucket)

Create a Bucket named `Bucket` (any region). Services reference `BUCKET`, `ENDPOINT`, `REGION`, `ACCESS_KEY_ID` and `SECRET_ACCESS_KEY` from it.

## Improvements over market templates

| # | Change | Why |
|---|---|---|
| 1 | Pinned `twentycrm/twenty:v2.45.8` (leader: `latest`) | Reproducible deploys; upgrades become a deliberate tag change. |
| 2 | Railway base Postgres `postgres-ssl:18` (leader: plain `postgres:16`, `PGHOST=db`) | Standard Railway variables, TLS, backups UI, draining on redeploy. |
| 3 | `ENCRYPTION_KEY` generated in addition to `APP_SECRET` | Twenty v2.5+ encrypts stored secrets with `ENCRYPTION_KEY`; setting it from day one avoids a later key rotation. |
| 4 | Migrations in the **pre-deploy** command with a Postgres wait loop | Safe with replicas, no crash-loop on a failed upgrade, server boots fast. |
| 5 | No volume on the server (leader mounts one at an unused path) | Server is stateless → zero-downtime deploys and horizontal scaling. |
| 6 | Redis `--maxmemory-policy noeviction` | Upstream requirement for BullMQ queues. |
| 7 | SMTP + Google + Microsoft variables with pre-filled callback URLs | Gmail / Outlook mail and calendar sync and real e-mail delivery are one form away. |
| 8 | Worker references every shared setting from `Twenty` | One place to edit; no drift between server and worker. |
| 9 | `LOGIC_FUNCTION_TYPE` exposed with a security note | Workflow code steps are off by default (upstream production default); enabling them is explicit. |
| 10 | Dropped obsolete/unused variables (`ENABLE_ALPINE_PRIVATE_NETWORKING`, `STORAGE_LOCAL_PATH`, `PGUSER_SUPERUSER`, public Redis URL/TCP proxy) | Less noise, no public Redis exposure. |

## First login

1. Open the Twenty public URL. The first visitor creates the first workspace and becomes its admin (`IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS` applies to later workspaces).
2. Invite teammates from **Settings → Members** (needs SMTP, otherwise the invite link is printed in the server logs because `EMAIL_DRIVER=LOGGER`).
3. Mail/calendar sync: create a Google Cloud OAuth client (Gmail API, Calendar API, People API) and/or a Microsoft Entra app, register the two callback URLs shown in the variables, fill the client ID/secret and set the matching `*_ENABLED` flags to `true`. Redeploy Twenty and Twenty Worker.

## Scaling

- **Twenty (server):** stateless — raise replicas in Railway (Settings → Replicas). Migrations stay single-run (pre-deploy).
- **Twenty Worker:** raise replicas for heavy mail/calendar sync or workflow load; BullMQ distributes jobs. Use `WORKER_ENABLED_QUEUES` / `WORKER_EXCLUDED_QUEUES` (Twenty variables) to dedicate workers to queues if needed.
- **Postgres:** vertical scaling; Railway can convert it to an HA cluster later.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month, Bucket $0.015/GB-month.

| Service | Idle RAM | Idle CPU | Typical RAM (5–25 users) | Typical CPU |
|---|---|---|---|---|
| Twenty | 550 MB | 0.02 vCPU | 900 MB | 0.15 vCPU |
| Twenty Worker | 400 MB | 0.02 vCPU | 700 MB | 0.15 vCPU |
| Postgres | 120 MB | 0.01 vCPU | 300 MB | 0.05 vCPU |
| Redis | 20 MB | <0.01 vCPU | 50 MB | 0.01 vCPU |
| **Total** | **≈1.1 GB** | **≈0.05 vCPU** | **≈1.95 GB** | **≈0.36 vCPU** |

- **Idle:** ≈ $11 RAM + $1 CPU + $0.2 volumes ≈ **$12/month**.
- **Typical:** ≈ $19.5 RAM + $7 CPU + $0.5 volumes (3 GB) + Bucket (5 GB, $0.08) ≈ **$27/month**.
- **Minimum plan:** Hobby. Each service fits Trial's 1 GB/service limit at idle, but server + worker exceed it during upgrades and sync bursts, and Trial credit does not cover a month.

## Verification checklist

- [ ] Pre-deploy log of **Twenty** shows `Database appears to be empty, running migrations` (first deploy) and `Successfully migrated DB!`; deployment then turns healthy on `/healthz`.
- [ ] Open the public URL, create the first workspace, create a Company and a Person.
- [ ] Upload an attachment on a record → the object appears in the Bucket (Railway → Bucket → Files) and downloads again. (Confirms the path-style S3 access Twenty uses works with the Bucket.)
- [ ] **Twenty Worker** logs show queue processing (e.g. after importing a CSV of contacts).
- [ ] Redeploy Twenty: data and attachments are still there; worker reconnects.
- [ ] Optional: set SMTP → send an invite; set Google OAuth → connect a Gmail account and see messages appear on the timeline.

## Open questions

- **Path-style S3:** Twenty's S3 driver forces path-style URLs; Railway documents virtual-hosted style for new buckets. The market leader uses the same combination with 660 active projects, so it is expected to work, but it was not verified on a fresh Bucket here (checklist item above).
- **Pre-deploy command parsing:** the pre-deploy command is a `/bin/sh -c "…"` string with nested single quotes; confirm Railway passes it unchanged (pattern also used by other templates).
- **Empty optional variables:** if Railway creates the blank optional variables as empty strings, Twenty treats them as unset for disabled features; values saved in the Admin Panel still take precedence.
- **`v2.46.0`** appeared on Docker Hub one day before the check; bump the tag once its GitHub release is published.
