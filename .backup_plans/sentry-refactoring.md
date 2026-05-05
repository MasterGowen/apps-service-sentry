# План рефакторинга Sentry blueprint

## 1. Текущее состояние

### Проблемы

- ❌ Отсутствует обязательное поле `container_name` в routing ([`service.yml:13`](../service.yml:13))
- ❌ Один контейнер без инфраструктуры (нет PostgreSQL, Redis)
- ❌ `build: .` без Dockerfile, пустая папка `src/`
- ❌ Лишняя сеть `servicenet`, проброс портов наружу
- ❌ Поле `maintainer` не из контракта платформы
- ❌ Нет worker и cron контейнеров для полноценной работы Sentry

### Текущая структура

```text
sentry/
├── service.yml           # Неполный манифест
├── docker-compose.yml    # Один контейнер, неправильная сеть
├── .env.example          # Минимальные переменные
├── .gitignore           # Минимальный
├── README.md            # Минимальный
└── src/                 # Пустая папка
    └── __init__.py
```text

## 2. Целевая архитектура

### Компоненты решения

```text
┌─────────────────────────────────────────────────────────┐
│  Caddy (platform_network)                               │
│  ├─ apps.openedu.urfu.ru/sentry → sentry-web:9000          │
└─────────────────────────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────┐
│  Sentry Services (platform_network + sentry_internal)   │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ sentry-web   │  │sentry-worker │  │ sentry-cron  │  │
│  │ (UI + API)   │  │(events queue)│  │ (cleanup)    │  │
│  │ Port: 9000   │  │              │  │              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │          │
│         └─────────────────┴─────────────────┘          │
│                           │                            │
│         ┌─────────────────┴─────────────────┐          │
│         ↓                                   ↓          │
│  ┌──────────────┐                   ┌──────────────┐   │
│  │  PostgreSQL  │                   │    Redis     │   │
│  │  Port: 5432  │                   │  Port: 6379  │   │
│  │  (metadata)  │                   │ (cache+queue)│   │
│  └──────────────┘                   └──────────────┘   │
└─────────────────────────────────────────────────────────┘
```text

### Сервисы (5 контейнеров)

1. **sentry-web** - основной контейнер (UI + API)
   - Образ: `sentry:24` (latest stable)
   - Порт: 9000 (внутренний)
   - Health: `/api/0/health/` endpoint

2. **sentry-worker** - обработка очереди событий
   - Образ: `sentry:24`
   - Команда: `run worker`
   - Без портов

3. **sentry-cron** - фоновые задачи (cleanup, etc)
   - Образ: `sentry:24`
   - Команда: `run cron`
   - Без портов

4. **postgres** - основная БД
   - Образ: `postgres:15-alpine`
   - Volume: `sentry-postgres`
   - Health: `pg_isready`

5. **redis** - кэш и очереди
   - Образ: `redis:7-alpine`
   - Volume: `sentry-redis`
   - Health: `redis-cli ping`

### Сети

- **platform_network** (external) - для интеграции с Caddy
- **sentry_internal** (internal) - для взаимодействия между компонентами

## 3. Детали реализации

### service.yml

```yaml
name: sentry
display_name: Sentry
version: 1.0.0
description: Система мониторинга и отслеживания ошибок Sentry (self-hosted)
type: docker-compose
visibility: internal

routing:
  - type: subfolder
    base_domain: apps.openedu.urfu.ru
    path: /sentry
    internal_port: 9000
    container_name: sentry-web    # ✅ КРИТИЧНО: обязательное поле
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
```text

### docker-compose.yml - Ключевые моменты

**Общие правила:**

- Все контейнеры в `platform_network` (для Caddy)
- БД и Redis дополнительно в `sentry_internal`
- Используются named volumes (не bind mounts)
- Health checks для критичных сервисов
- `restart: unless-stopped` для всех

**Переменные окружения (общие для sentry-*):**

```yaml
SENTRY_SECRET_KEY: ${SENTRY_SECRET_KEY}
SENTRY_POSTGRES_HOST: postgres
SENTRY_POSTGRES_PORT: 5432
SENTRY_DB_NAME: ${SENTRY_DB_NAME:-sentry}
SENTRY_DB_USER: ${SENTRY_DB_USER}
SENTRY_DB_PASSWORD: ${SENTRY_DB_PASSWORD}
SENTRY_REDIS_HOST: redis
SENTRY_REDIS_PORT: 6379
```text

**Depends_on с условиями:**

```yaml
depends_on:
  postgres:
    condition: service_healthy
  redis:
    condition: service_healthy
```text

### .env.example

Основные переменные:

- `SENTRY_SECRET_KEY` - генерируется при установке
- `SENTRY_DB_USER` / `SENTRY_DB_PASSWORD` - учетные данные PostgreSQL
- `SENTRY_DB_NAME` - имя БД (по умолчанию: sentry)
- `POSTGRES_PASSWORD` - пароль для postgres
- `SENTRY_MAIL_*` - опционально для email уведомлений

### .gitignore

Добавить:

```text
.env
service.local.yml
volumes/
backups/
*.db
combined.md
```text

### README.md

Разделы:

1. Описание (что это за сервис)
2. Требования (минимум ресурсов: ~2GB RAM)
3. Быстрый старт (4 шага)
4. Первичная настройка (создание суперпользователя)
5. Переменные окружения
6. Управление через платформу
7. Troubleshooting

### service.local.yml (пример)

Для переопределения параметров в dev/staging:

```yaml
routing:
  - base_domain: apps.dev.example.com
health:
  interval: 60s
```text

## 4. Порядок внедрения

### Этап 1: Подготовка

- Удалить папку `src/` (не нужна для официальных образов)
- Обновить `.gitignore`

### Этап 2: Конфигурация

- Исправить `service.yml` (добавить container_name, убрать maintainer)
- Создать полный `.env.example`
- Создать `service.local.yml` example

### Этап 3: Инфраструктура

- Переписать `docker-compose.yml` с 5 сервисами
- Добавить volumes, networks, health checks
- Настроить depends_on с условиями

### Этап 4: Документация

- Обновить `README.md` с инструкциями
- Добавить примеры команд

### Этап 5: Git workflow

- Создать feature ветку
- Коммиты по логическим блокам
- Создать PR для review

## 5. Соответствие контракту платформы

✅ **Обязательные файлы:**

- `service.yml` - минимально необходимые поля
- `docker-compose.yml` - тип docker-compose
- `.env.example` - шаблон переменных
- `service.local.yml` example - для переопределений

✅ **service.yml - только поддерживаемые поля:**

- Meta: name, display_name, version, description, type, visibility
- Routing: с обязательным container_name
- Health: enabled, endpoint, interval, timeout, retries
- Backup: отключен (enabled: false)

✅ **docker-compose.yml - требования:**

- `container_name: sentry-web` совпадает с routing
- Сеть `platform_network` (external: true)
- Без пробоса портов наружу (только internal)

❌ **НЕ включать (не поддерживается платформой):**

- resources, dependencies, logging
- secrets (кроме через .env)
- hooks, notifications (кроме базового telegram)
- Prometheus, Grafana, Loki, Restic

## 6. Минимализм vs Функциональность

### ✅ Включено (минимум для работы Sentry)

- PostgreSQL (необходима для metadata)
- Redis (необходим для очередей и кэша)
- Worker (необходим для обработки событий)
- Cron (необходим для cleanup задач)

### ❌ НЕ включено (оверхед)

- Clickhouse (отказ по требованию)
- MemCached (Redis справляется)
- Kafka (избыточно для низкой нагрузки)
- Relay (не нужен для internal usage)
- Nginx (Caddy уже есть на платформе)

### Оценка ресурсов

- **RAM**: ~1.5-2GB (все контейнеры)
- **Disk**: ~5-10GB (с учетом роста БД)
- **CPU**: минимально (low traffic)

## 7. Итоговая структура проекта

```text
sentry/
├── service.yml                 # ✅ Исправленный манифест
├── docker-compose.yml          # ✅ 5 сервисов + volumes + networks
├── .env.example                # ✅ Полный набор переменных
├── .gitignore                  # ✅ Дополнен
├── README.md                   # ✅ Подробные инструкции
├── service.local.yml.example   # ✅ Пример для переопределений
└── plans/
    └── sentry-refactoring.md   # Этот документ
```text

## 8. Проверка готовности

После рефакторинга проверить:

- [ ] `service.yml` содержит `container_name` в routing
- [ ] `docker-compose.yml` использует правильную сеть `platform_network`
- [ ] Нет проброса портов наружу
- [ ] Health checks настроены для всех критичных сервисов
- [ ] `.env.example` содержит все необходимые переменные
- [ ] README содержит инструкции по первичной настройке
- [ ] `.gitignore` содержит `.env`, `service.local.yml`
- [ ] Удалена папка `src/`
- [ ] Нет лишних полей в `service.yml` (maintainer, resources, etc)

---

**Следующий шаг**: Согласование плана → переключение в Code mode → реализация по todo list → git workflow (feature branch + PR)
