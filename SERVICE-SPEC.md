# Спецификация сервиса Sentry

## Конфигурация сервиса

### service.yml

```yaml
name: sentry
display_name: Sentry
version: 1.0.0
description: Система мониторинга и отслеживания ошибок Sentry (self-hosted, минимальная конфигурация)
type: docker-compose
visibility: internal

routing:
  - type: subfolder
    base_domain: apps.openedu.urfu.ru
    path: /sentry
    internal_port: 9000
    container_name: sentry-web
    strip_prefix: true

health:
  enabled: true
  endpoint: /api/0/health/
  interval: 30s
  timeout: 10s
  retries: 3

backup:
  enabled: false
  schedule: "0 2 * * *"
  retention: 7
```

Обязательное поле `container_name` необходимо для интеграции с Caddy через `platform_network`.

### docker-compose.yml

```yaml
version: "3.8"

services:
  sentry-web:
    image: sentry:24
    container_name: sentry-web
    restart: unless-stopped
    networks:
      - platform_network
      - sentry_internal
    environment:
      SENTRY_SECRET_KEY: ${SENTRY_SECRET_KEY}
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_POSTGRES_PORT: 5432
      SENTRY_DB_NAME: ${SENTRY_DB_NAME:-sentry}
      SENTRY_DB_USER: ${SENTRY_DB_USER}
      SENTRY_DB_PASSWORD: ${SENTRY_DB_PASSWORD}
      SENTRY_REDIS_HOST: redis
      SENTRY_REDIS_PORT: 6379
      SENTRY_WEB_HOST: 0.0.0.0
      SENTRY_WEB_PORT: 9000
      SENTRY_MAIL_HOST: ${SENTRY_MAIL_HOST:-}
      SENTRY_MAIL_PORT: ${SENTRY_MAIL_PORT:-}
      SENTRY_MAIL_USERNAME: ${SENTRY_MAIL_USERNAME:-}
      SENTRY_MAIL_PASSWORD: ${SENTRY_MAIL_PASSWORD:-}
      SENTRY_MAIL_USE_TLS: ${SENTRY_MAIL_USE_TLS:-false}
      SENTRY_SERVER_EMAIL: ${SENTRY_SERVER_EMAIL:-}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "--quiet", "http://localhost:9000/api/0/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  sentry-worker:
    image: sentry:24
    container_name: sentry-worker
    restart: unless-stopped
    command: run worker
    networks:
      - sentry_internal
    environment:
      SENTRY_SECRET_KEY: ${SENTRY_SECRET_KEY}
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_POSTGRES_PORT: 5432
      SENTRY_DB_NAME: ${SENTRY_DB_NAME:-sentry}
      SENTRY_DB_USER: ${SENTRY_DB_USER}
      SENTRY_DB_PASSWORD: ${SENTRY_DB_PASSWORD}
      SENTRY_REDIS_HOST: redis
      SENTRY_REDIS_PORT: 6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  sentry-cron:
    image: sentry:24
    container_name: sentry-cron
    restart: unless-stopped
    command: run cron
    networks:
      - sentry_internal
    environment:
      SENTRY_SECRET_KEY: ${SENTRY_SECRET_KEY}
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_POSTGRES_PORT: 5432
      SENTRY_DB_NAME: ${SENTRY_DB_NAME:-sentry}
      SENTRY_DB_USER: ${SENTRY_DB_USER}
      SENTRY_DB_PASSWORD: ${SENTRY_DB_PASSWORD}
      SENTRY_REDIS_HOST: redis
      SENTRY_REDIS_PORT: 6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:15-alpine
    container_name: sentry-postgres
    restart: unless-stopped
    networks:
      - sentry_internal
    environment:
      POSTGRES_USER: ${SENTRY_DB_USER}
      POSTGRES_PASSWORD: ${SENTRY_DB_PASSWORD}
      POSTGRES_DB: ${SENTRY_DB_NAME:-sentry}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - sentry-postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${SENTRY_DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: sentry-redis
    restart: unless-stopped
    networks:
      - sentry_internal
    volumes:
      - sentry-redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  platform_network:
    external: true
    name: platform_network
  sentry_internal:
    name: sentry_internal
    internal: true

volumes:
  sentry-postgres:
    name: sentry-postgres
  sentry-redis:
    name: sentry-redis
```

Ключевые характеристики:

- Пять контейнеров: web, worker, cron, postgres, redis
- Только sentry-web подключён к `platform_network`
- Все сервисы используют внутреннюю сеть `sentry_internal`
- Используются именованные объёмы (named volumes)
- Health check с условием `service_healthy`
- `start_period: 60s` для sentry-web (учёт времени инициализации)
- Нет проброса портов наружу

## Переменные окружения

### .env.example

```bash
# SENTRY CONFIGURATION
SENTRY_SECRET_KEY=change-me-to-random-50-char-string

# DATABASE (PostgreSQL)
SENTRY_DB_USER=sentry
SENTRY_DB_PASSWORD=change-me-strong-password
SENTRY_DB_NAME=sentry

# EMAIL (опционально)
SENTRY_MAIL_HOST=
SENTRY_MAIL_PORT=587
SENTRY_MAIL_USERNAME=
SENTRY_MAIL_PASSWORD=
SENTRY_MAIL_USE_TLS=true
SENTRY_SERVER_EMAIL=sentry@example.com
```

### .gitignore

```gitignore
# Environment
.env
*.local.yml

# Volumes
volumes/
data/

# Backups
backups/

# Database
*.db
*.sqlite

# Documentation
combined.md

# Logs
*.log
logs/
```

## Локальные переопределения

### service.local.yml.example

```yaml
# Пример локальных переопределений
# Скопируйте в service.local.yml

routing:
  - type: subfolder
    base_domain: apps.dev.openedu.urfu.ru
    path: /sentry

health:
  interval: 60s
  timeout: 15s

backup:
  enabled: true
  schedule: "0 3 * * *"
  retention: 14
  databases:
    - type: postgres
      container: sentry-postgres
      database: sentry
```

Файл объединяется с `service.yml` платформой при деплое.
