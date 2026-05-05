# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Type

Infrastructure-only (Docker Compose). No source code, tests, or build steps. Uses official Sentry 24 images.

## Deployment

- Deploy: `platform deploy sentry` (from master service directory)
- Platform merges `service.local.yml` overrides into `service.yml`
- Requires external network `platform_network` (created by platform)

## Critical Configuration

- `service.yml` routing REQUIRES `container_name: sentry-web` (platform contract)
- Health endpoint: `/api/0/health/` (not `/health`)
- First deploy requires manual setup:
  - `docker exec -it sentry-web sentry upgrade --noinput`
  - `docker exec -it sentry-web sentry createuser --email admin@example.com --password <pw> --superuser --no-input`

## Environment

- Copy `.env.example` to `.env` and generate `SENTRY_SECRET_KEY` with `openssl rand -base64 50`
- Never commit `.env`, `service.local.yml`, or volume data
- Email (SMTP) config is optional

## Architecture

- 5 containers: web (port 9000), worker, cron, postgres 15, redis 7
- 2 networks: `platform_network` (external, for Caddy routing), `sentry_internal` (isolated backend)
- No Clickhouse (minimal config), backup disabled by default
