# Deploy and Host Twenty with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/twenty-v2-crm-sales-workspace)

Twenty is an open-source CRM for managing companies, people, opportunities and workflows, with a fast spreadsheet-style interface and a full API. This community template runs Twenty v2 on Railway with a dedicated background worker, Postgres, Redis and a Railway Bucket for files, so your customer data stays in your own project.

## About Hosting Twenty

Twenty is a NestJS server that also serves the web app, plus a worker process that handles background jobs: e-mail and calendar synchronisation, workflow runs, imports and webhooks. It needs Postgres for data, Redis for cache and job queues, and file storage for attachments. This template wires those pieces together on Railway's private network, keeps attachments in a Railway Bucket so the server stays stateless, runs database migrations as a pre-deploy step, and generates every secret at deploy time. SMTP and Google or Microsoft mail/calendar sync are prepared as optional variables and can also be set later in Twenty's Admin Panel.

## Common Use Cases

- Running a sales pipeline (companies, contacts, opportunities) for a team without per-seat licence fees
- Syncing Gmail or Microsoft 365 mail and calendars into a shared customer timeline
- Automating follow-ups and data updates with Twenty workflows and webhooks
- Building internal tools on top of Twenty's GraphQL and REST APIs

## Dependencies for Twenty Hosting

- Twenty server and Twenty worker (`twentycrm/twenty`, pinned)
- PostgreSQL 15 or newer (Railway Postgres 18)
- Redis (cache and BullMQ queues)
- S3-compatible object storage (Railway Bucket)

### Deployment Dependencies

- Twenty self-hosting guide: https://docs.twenty.com/developers/self-host/capabilities/setup
- Twenty upgrade guide: https://docs.twenty.com/developers/self-host/capabilities/upgrade-guide
- Twenty source and releases: https://github.com/twentyhq/twenty
- Railway Buckets: https://docs.railway.com/storage-buckets

### Implementation Details

| Service | Image | Role |
|---|---|---|
| Twenty | `twentycrm/twenty:v2.45.8` | API and web app on the public domain; runs migrations in its pre-deploy step |
| Twenty Worker | `twentycrm/twenty:v2.45.8` | Background jobs (`yarn worker:prod`) |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl:18` | Database (volume) |
| Redis | `redis:8.2` | Cache and job queues (volume, `noeviction`) |
| Bucket | Railway Bucket | Attachments and uploaded files |

**First login:** open the public URL and create your workspace; the first account becomes the workspace admin. Without SMTP, invitation e-mails are written to the Twenty logs. To enable e-mail or Gmail/Outlook sync, fill the `EMAIL_SMTP_*`, `AUTH_GOOGLE_*` or `AUTH_MICROSOFT_*` variables on the Twenty service (the worker reads them by reference) and redeploy, or set them in Settings → Admin Panel.

**Scaling:** the server keeps no local state, so you can add replicas to Twenty; add replicas to Twenty Worker for heavy sync or workflow volume.

**Versions:** both Twenty services use the same pinned tag. To upgrade, change the tag on both, back up Postgres, and redeploy; migrations run once in the pre-deploy step.

### Why Deploy Twenty on Railway?

Railway runs the server, worker, database, cache and bucket as one project with private networking, managed volumes and usage-based billing. You get a production-shaped Twenty install without maintaining servers, and you can scale each part independently as your team grows.
