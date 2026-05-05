# Code Mode - Project Specific Rules

## Docker Compose

- Use version "3.8" (required for healthcheck `start_period`)
- Container names must follow pattern: `sentry-{service}` (e.g., `sentry-web`, `sentry-postgres`)
- Always use `restart: unless-stopped` (not `always`)

## Environment Variables

- Reference via `${VAR:-default}` syntax in docker-compose.yml
- Never hardcode secrets - use `.env` with `.env.example` as template
- `SENTRY_SECRET_KEY` must be 50+ characters (generated via `openssl rand -base64 50`)

## File Conventions

- Indentation: 2 spaces in YAML files
- Comments: use `# ==========` style section dividers in docker-compose.yml
- Health checks: use `wget --spider` for HTTP, `pg_isready` for Postgres, `redis-cli ping` for Redis
