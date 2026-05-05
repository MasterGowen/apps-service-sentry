# Спецификация реализации Sentry Blueprint

## Обзор изменений

Трансформация "рахитичного" блюпринта с одним контейнером в полноценное self-hosted Sentry решение с минимальным оверхедом.

## Задачи по приоритету

### 🔴 Критично (блокирует работу платформы)

#### 1. service.yml - исправление критических ошибок

**Файл**: `service.yml`

**Проблемы**:

- ❌ Отсутствует `container_name` в routing (обязательное поле)
- ❌ Поле `maintainer` не из контракта платформы
- ❌ Health endpoint должен быть `/api/0/health/` (стандартный для Sentry)

**Изменения**:

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
    container_name: sentry-web        # ⚠️ ДОБАВИТЬ - обязательно!
    strip_prefix: true

health:
  enabled: true
  endpoint: /api/0/health/            # Изменить с /healthz
  interval: 30s
  timeout: 10s                        # Добавить timeout
  retries: 3                          # Добавить retries

backup:
  enabled: false
  schedule: "0 2 * * *"
  retention: 7
```

**Удалить**: поле `maintainer` (не входит в контракт)

---

#### 2. docker-compose.yml - полная переработка

**Файл**: `docker-compose.yml`

**Текущие проблемы**:

- ❌ Один контейнер (нужно 5)
- ❌ `build: .` без Dockerfile
- ❌ Лишняя сеть `servicenet`
- ❌ Проброс портов наружу `8000:8000`
- ❌ Нет зависимостей (postgres, redis)

**Новая структура**:

```yaml
version: "3.8"

services:
  # ============================================
  # 1. SENTRY WEB - основной UI + API
  # ============================================
  sentry-web:
    image: sentry:24
    container_name: sentry-web
    restart: unless-stopped
    networks:
      - platform_network
      - sentry_internal
    environment:
      # Core
      SENTRY_SECRET_KEY: ${SENTRY_SECRET_KEY}
      # Database
      SENTRY_POSTGRES_HOST: postgres
      SENTRY_POSTGRES_PORT: 5432
      SENTRY_DB_NAME: ${SENTRY_DB_NAME:-sentry}
      SENTRY_DB_USER: ${SENTRY_DB_USER}
      SENTRY_DB_PASSWORD: ${SENTRY_DB_PASSWORD}
      # Redis
      SENTRY_REDIS_HOST: redis
      SENTRY_REDIS_PORT: 6379
      # Service
      SENTRY_WEB_HOST: 0.0.0.0
      SENTRY_WEB_PORT: 9000
      # Mail (optional)
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

  # ============================================
  # 2. SENTRY WORKER - обработка событий
  # ============================================
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

  # ============================================
  # 3. SENTRY CRON - фоновые задачи
  # ============================================
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

  # ============================================
  # 4. POSTGRESQL - основная БД
  # ============================================
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

  # ============================================
  # 5. REDIS - кэш и очереди
  # ============================================
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

# ============================================
# NETWORKS
# ============================================
networks:
  platform_network:
    external: true
    name: platform_network
  sentry_internal:
    name: sentry_internal
    internal: true

# ============================================
# VOLUMES
# ============================================
volumes:
  sentry-postgres:
    name: sentry-postgres
  sentry-redis:
    name: sentry-redis
```

**Ключевые моменты**:

- ✅ 5 контейнеров: web, worker, cron, postgres, redis
- ✅ Только sentry-web в `platform_network`
- ✅ Все в `sentry_internal` для межсервисного взаимодействия
- ✅ Named volumes (не bind mounts)
- ✅ Health checks с `condition: service_healthy`
- ✅ `start_period: 60s` для sentry-web (долгий старт)
- ✅ Нет проброса портов наружу

---

### 🟡 Важно (конфигурация)

#### 3. .env.example - полный набор переменных

**Файл**: `.env.example`

```bash
# ==============================================
# SENTRY CONFIGURATION
# ==============================================

# Секретный ключ (генерируется при установке)
# Сгенерировать: openssl rand -base64 50
SENTRY_SECRET_KEY=change-me-to-random-50-char-string

# ==============================================
# DATABASE (PostgreSQL)
# ==============================================
SENTRY_DB_USER=sentry
SENTRY_DB_PASSWORD=change-me-strong-password
SENTRY_DB_NAME=sentry

# ==============================================
# EMAIL (опционально)
# ==============================================
# Для email уведомлений о новых ошибках
SENTRY_MAIL_HOST=
SENTRY_MAIL_PORT=587
SENTRY_MAIL_USERNAME=
SENTRY_MAIL_PASSWORD=
SENTRY_MAIL_USE_TLS=true
SENTRY_SERVER_EMAIL=sentry@example.com

# ==============================================
# ПРИМЕЧАНИЯ
# ==============================================
# 1. Скопируйте этот файл в .env
# 2. Измените все значения "change-me-*"
# 3. Для генерации SENTRY_SECRET_KEY используйте:
#    openssl rand -base64 50
# 4. НИКОГДА не коммитьте .env в git!
```

---

#### 4. .gitignore - дополнить

**Файл**: `.gitignore`

**Добавить**:

```gitignore
# Environment
.env
*.local.yml

# Volumes (если используются bind mounts)
volumes/
data/

# Backups
backups/

# Database
*.db
*.sqlite

# Documentation (auto-generated)
combined.md

# Logs
*.log
logs/
```

---

#### 5. service.local.yml.example - пример переопределений

**Файл**: `service.local.yml.example` (новый)

```yaml
# Пример локальных переопределений для service.yml
# Скопируйте в service.local.yml и измените под свои нужды
# Этот файл мержится поверх service.yml платформой

# Переопределение домена для dev/staging
routing:
  - type: subfolder
    base_domain: apps.dev.openedu.urfu.ru
    path: /sentry

# Более редкие health checks для экономии ресурсов
health:
  interval: 60s
  timeout: 15s

# Включить backup для production
backup:
  enabled: true
  schedule: "0 3 * * *"
  retention: 14
  databases:
    - type: postgres
      container: sentry-postgres
      database: sentry
```

---

### 🟢 Документация

#### 6. README.md - подробные инструкции

**Файл**: `README.md`

```markdown
# Sentry - Система мониторинга ошибок

Self-hosted Sentry для отслеживания и мониторинга ошибок приложений (минимальная конфигурация без Clickhouse).

## 📋 Описание

Sentry (sentry.io) - открытая система для сбора, агрегации и анализа ошибок в приложениях. Поддерживает 100+ языков и фреймворков.

**Состав**:
- Sentry Web (UI + API)
- Sentry Worker (обработка событий)
- Sentry Cron (фоновые задачи)
- PostgreSQL 15 (метаданные и события)
- Redis 7 (очереди и кэш)

**Ресурсы**: ~1.5-2GB RAM, ~5-10GB Disk

## 🚀 Быстрый старт

### 1. Подготовка переменных окружения

```bash
# Скопировать шаблон
cp .env.example .env

# Сгенерировать секретный ключ
openssl rand -base64 50

# Отредактировать .env и вставить ключ
nano .env
```

**Обязательно изменить**:

- `SENTRY_SECRET_KEY` - на сгенерированный ключ
- `SENTRY_DB_PASSWORD` - на сильный пароль

### 2. Развертывание через платформу

```bash
# Из директории master service
platform deploy sentry
```

Платформа автоматически:

- Создаст контейнеры
- Настроит Caddy routing
- Запустит health monitoring

### 3. Инициализация БД и создание суперпользователя

После первого запуска выполнить:

```bash
# Применить миграции (автоматически при первом запуске)
docker exec -it sentry-web sentry upgrade --noinput

# Создать суперпользователя
docker exec -it sentry-web sentry createuser \
  --email admin@example.com \
  --password admin123 \
  --superuser \
  --no-input
```

### 4. Доступ к интерфейсу

- URL: `https://apps.openedu.urfu.ru/sentry`
- Login: email из команды createuser
- Password: пароль из команды createuser

После первого входа:

1. Создайте организацию
2. Создайте проект (выберите платформу)
3. Получите DSN для интеграции с приложениями

## 🔧 Переменные окружения

| Переменная | Описание | Обязательно |
|------------|----------|-------------|
| `SENTRY_SECRET_KEY` | Секретный ключ (50+ символов) | ✅ |
| `SENTRY_DB_USER` | Пользователь PostgreSQL | ✅ |
| `SENTRY_DB_PASSWORD` | Пароль PostgreSQL | ✅ |
| `SENTRY_DB_NAME` | Имя БД (default: sentry) | ❌ |
| `SENTRY_MAIL_HOST` | SMTP сервер | ❌ |
| `SENTRY_MAIL_PORT` | SMTP порт (default: 587) | ❌ |
| `SENTRY_MAIL_USERNAME` | SMTP логин | ❌ |
| `SENTRY_MAIL_PASSWORD` | SMTP пароль | ❌ |
| `SENTRY_SERVER_EMAIL` | Email отправителя | ❌ |

## 📊 Управление

### Логи

```bash
# Все сервисы
platform logs sentry

# Конкретный контейнер
docker logs sentry-web -f
docker logs sentry-worker -f
```

### Остановка/запуск

```bash
platform stop sentry
platform start sentry
```

### Статус

```bash
platform status sentry
docker ps | grep sentry
```

### Очистка старых событий

Sentry автоматически чистит старые события через sentry-cron. Настройка в UI:

- Settings → Data → Data Scrubbing
- Retention: настройка хранения событий

## 🔍 Интеграция с приложениями

### Python

```bash
pip install sentry-sdk
```

```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://PUBLIC_KEY@apps.openedu.urfu.ru/sentry/PROJECT_ID",
    traces_sample_rate=1.0,
)
```

### JavaScript

```bash
npm install @sentry/browser
```

```javascript
import * as Sentry from "@sentry/browser";

Sentry.init({
  dsn: "https://PUBLIC_KEY@apps.openedu.urfu.ru/sentry/PROJECT_ID",
});
```

DSN получается в: Project Settings → Client Keys (DSN)

## 🧩 Архитектура

```text
apps.openedu.urfu.ru/sentry (Caddy)
           ↓
    sentry-web :9000
         ↓     ↓
    postgres  redis
         ↓     ↓
    sentry-worker
    sentry-cron
```

- **platform_network**: sentry-web (для Caddy)
- **sentry_internal**: все сервисы (изолированно)

## 🐛 Troubleshooting

### Сервис не стартует

```bash
# Проверить логи
docker logs sentry-web

# Частые причины:
# 1. SENTRY_SECRET_KEY не задан или короткий
# 2. Нет миграций - выполнить: docker exec -it sentry-web sentry upgrade
# 3. PostgreSQL не готов - проверить: docker logs sentry-postgres
```

### Health check failed

```bash
# Проверить health endpoint
docker exec -it sentry-web wget -O- http://localhost:9000/api/0/health/

# Если 502 Bad Gateway:
# 1. Проверить, что sentry-web запущен: docker ps
# 2. Проверить логи: docker logs sentry-web
# 3. Возможно долгий старт - подождать 1-2 минуты
```

### Worker не обрабатывает события

```bash
# Проверить worker
docker logs sentry-worker

# Проверить Redis
docker exec -it sentry-redis redis-cli ping
# Должен вернуть: PONG

# Перезапустить worker
docker restart sentry-worker
```

### Ошибки БД

```bash
# Проверить PostgreSQL
docker exec -it sentry-postgres pg_isready -U sentry

# Применить миграции
docker exec -it sentry-web sentry upgrade

# Backup БД (ручной)
docker exec -it sentry-postgres pg_dump -U sentry sentry > sentry_backup.sql
```

## 📝 Примечания

- Clickhouse НЕ включен (режим минимального оверхеда)
- Backup отключен (если нужен - см. service.local.yml.example)
- Email уведомления опциональны (настраиваются через SENTRY_MAIL_*)
- Первый запуск может занять 1-2 минуты (инициализация БД)

## 📚 Дополнительно

- [Официальная документация Sentry](https://docs.sentry.io/)
- [Self-hosted Guide](https://develop.sentry.dev/self-hosted/)
- [SDK Documentation](https://docs.sentry.io/platforms/)

```

---

### 🔵 Финальные действия

#### 7. Удаление папки src/

**Действие**: удалить директорию `src/` полностью (не нужна для официальных образов)

```bash
rm -rf src/
```

---

## Git Workflow

### Создание feature branch

```bash
git checkout -b feature/sentry-blueprint-refactor
```

### Коммиты по этапам

```bash
# После удаления src/ и обновления .gitignore
git add .gitignore
git rm -r src/
git commit -m "chore: remove unused src/ directory, update .gitignore"

# После исправления service.yml
git add service.yml
git commit -m "fix: add required container_name in routing, update health endpoint"

# После создания docker-compose.yml
git add docker-compose.yml
git commit -m "feat: implement full Sentry infrastructure with 5 services"

# После создания .env.example и примеров
git add .env.example service.local.yml.example
git commit -m "feat: add comprehensive environment variables and local overrides example"

# После обновления документации
git add README.md plans/
git commit -m "docs: add detailed README and architecture plans"
```

### Создание Pull Request

```bash
git push origin feature/sentry-blueprint-refactor
```

Далее создать PR через интерфейс Git платформы с описанием:

```text
## Рефакторинг Sentry blueprint

### Изменения:
- ✅ Исправление критических ошибок в service.yml (container_name)
- ✅ Полная инфраструктура: web, worker, cron, postgres, redis
- ✅ Корректные сети (platform_network + sentry_internal)
- ✅ Health checks для всех важных сервисов
- ✅ Подробная документация и примеры

### Результат:
Полноценное self-hosted Sentry решение с минимальным оверхедом (~2GB RAM)
без Clickhouse, соответствующее контракту платформы автодеплоя.

### Проверено:
- [ ] service.yml валидируется платформой
- [ ] docker-compose up успешно запускает все 5 сервисов
- [ ] Health endpoints отвечают корректно
- [ ] Caddy routing работает через platform_network
```

---

## Чеклист перед PR

- [ ] Удалена папка `src/`
- [ ] `service.yml` содержит `container_name: sentry-web`
- [ ] `docker-compose.yml` использует правильную сеть `platform_network`
- [ ] Нет проброса портов наружу в docker-compose
- [ ] Health checks настроены для postgres, redis, sentry-web
- [ ] `.env.example` содержит все переменные с описанием
- [ ] `.gitignore` дополнен (service.local.yml, volumes/, backups/)
- [ ] README содержит troubleshooting секцию
- [ ] Создан `service.local.yml.example`
- [ ] Нет лишних полей в `service.yml` (maintainer удален)
- [ ] Все коммиты имеют осмысленные сообщения
- [ ] `.env` НЕ закоммичен

---

## Порядок реализации (для Code mode)

1. **Подготовка**: удаление src/, обновление .gitignore
2. **Конфигурация**: исправление service.yml
3. **Инфраструктура**: новый docker-compose.yml
4. **Переменные**: .env.example и service.local.yml.example
5. **Документация**: README.md
6. **Git**: коммиты и PR

**Критично**: каждый файл должен быть валидным YAML/Markdown перед коммитом!
