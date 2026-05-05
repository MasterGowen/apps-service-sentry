# Architect Mode - Project Specific Rules

## Platform Contract (CRITICAL)

- `service.yml` routing MUST specify `container_name: sentry-web` (platform requirement)
- Health endpoint: `/api/0/health/` (NOT `/health` or `/_health`)
- External network `platform_network` must exist before deploy (created by platform)

## Network Architecture

- `platform_network`: external network for Caddy routing (sentry-web only)
- `sentry_internal`: internal network for backend services (postgres, redis, worker, cron)
- Only `sentry-web` exposes ports (9000), others communicate via internal network

## Container Design

- Single image `sentry:24` for web/worker/cron (different `command:`)
- No Clickhouse (minimal config by design)
- Backup disabled by default (data non-critical for monitoring)
