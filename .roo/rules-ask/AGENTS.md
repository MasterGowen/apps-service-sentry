# Ask Mode - Project Context

## Project Nature

- Infrastructure-only project (NO application code)
- Uses official Sentry 24 Docker images
- Deployed via platform automation (`platform deploy sentry`)

## Key Facts

- 5 containers: web (:9000), worker, cron, postgres 15, redis 7
- URL: `https://apps.openedu.urfu.ru/sentry`
- Resources: ~1.5-2GB RAM, ~5-10GB disk
- Optional: Email notifications (SMTP config)

## Common Questions

- "Why no Clickhouse?": Minimal config for basic error tracking
- "Why no source code?": Uses official images, configuration only
- "Where are volumes?": Named volumes `sentry-postgres`, `sentry-redis` (managed by Docker)
