# NOTICE — Twenty v2 CRM Sales Workspace

## Software deployed (not redistributed)

| Component | Source | License |
|---|---|---|
| Twenty | Docker image `twentycrm/twenty:v2.45.8` from https://github.com/twentyhq/twenty | AGPL-3.0 for the core (see the repository `LICENSE`; some enterprise files carry a separate commercial license) |
| PostgreSQL | Railway base image `ghcr.io/railwayapp-templates/postgres-ssl:18` | PostgreSQL License |
| Redis | `redis:8.2` (Railway base definition) | Redis 8 tri-license (RSALv2 / SSPLv1 / AGPLv3) |

The template references these images; it does not copy or modify their code.

## Upstream files checked (2026-10-10)

- `twentyhq/twenty` → `packages/twenty-docker/docker-compose.yml` and `packages/twenty-docker/.env.example` (branch `main`), for services, volumes and variables.
- `twentyhq/twenty` → `packages/twenty-docker/twenty/entrypoint.sh` and `Dockerfile` at tag `twenty/v2.45.6`, for the migration flow reused by the pre-deploy command.
- `twentyhq/twenty` → `packages/twenty-server/src/engine/core-modules/twenty-config/config-variables.ts` (branch `main`), for variable names, defaults and enum values (`S_3`, `LOGGER`, `DISABLED`).
- Docs: https://docs.twenty.com/developers/self-host/capabilities/setup and …/upgrade-guide (OAuth callback paths, `ENCRYPTION_KEY`, cross-version upgrades).

## Reused material

- **Railway base definitions** `templates/_railway-base/postgres.json` and `redis.json` (provided in this repository): copied as instructed by STANDARD.md; one flag (`--maxmemory-policy noeviction`, from Twenty's upstream compose) appended to the Redis start command.
- **Market template `nAL3hA` "Twenty CRM"** (Railway marketplace, no license): used as the structural starting point (server + worker split, Bucket wiring, `?family=0` Redis suffix, `/healthz`). No README text or descriptions were copied; all prose and variable descriptions in this package are original.
- No `_upstream` repositories exist for this app.
