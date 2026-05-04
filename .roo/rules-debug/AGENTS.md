# Debug Mode - Project Specific Rules

## Deployment Issues
- First deploy requires manual DB init: `docker exec -it sentry-web sentry upgrade --noinput`
- Superuser creation: `docker exec -it sentry-web sentry createuser --email ... --superuser --no-input`
- Containers may take 1-2 minutes to start (Postgres init + Sentry migrations)

## Health Checks
- Web: `http://localhost:9000/api/0/health/` (returns 200 when ready)
- Postgres: `pg_isready -U ${SENTRY_DB_USER}` (via docker healthcheck)
- Redis: `redis-cli ping` (expects PONG)

## Log Locations
- All services: `platform logs sentry` or `docker logs <container-name> -f`
- Sentry web specific: `docker exec -it sentry-web sentry config list` (debug config)
