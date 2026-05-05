# Sentry — Система мониторинга ошибок

Self-hosted Sentry для отслеживания и мониторинга ошибок приложений (минимальная конфигурация без Clickhouse).

## 📋 Описание

Sentry ([sentry.io](https://sentry.io)) — открытая система для сбора, агрегации и анализа ошибок в приложениях. Поддерживает 100+ языков и фреймворков.

**Состав**:

- **Sentry Web** — UI + API (порт 9000)
- **Sentry Worker** — обработка событий
- **Sentry Cron** — фоновые задачи (cleanup, статистика)
- **PostgreSQL 15** — метаданные и события
- **Redis 7** — очереди и кэш

**Ресурсы**: ~1.5–2 GB RAM, ~5–10 GB Disk

## 🚀 Быстрый старт

### 1. Подготовка переменных окружения

```bash
# Скопировать шаблон
cp .env.example .env

# Сгенерировать секретный ключ
openssl rand -base64 50

# Отредактировать .env и вставить ключ
nano .env
```text

**Обязательно изменить**:
- `SENTRY_SECRET_KEY` — на сгенерированный ключ (50+ символов)
- `SENTRY_DB_PASSWORD` — на сильный пароль

### 2. Развертывание через платформу

```bash
# Из директории master service
platform deploy sentry
```text

Платформа автоматически:
- Создаст контейнеры
- Настроит Caddy routing
- Запустит health monitoring

### 3. Инициализация БД и создание суперпользователя

После первого запуска выполнить:

```bash
# Применить миграции
docker exec -it sentry-web sentry upgrade --noinput

# Создать суперпользователя
docker exec -it sentry-web sentry createuser \
  --email admin@example.com \
  --password admin123 \
  --superuser \
  --no-input
```text

### 4. Доступ к интерфейсу

- **URL**: `https://apps.openedu.urfu.ru/sentry`
- **Login**: email из команды `createuser`
- **Password**: пароль из команды `createuser`

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
| `SENTRY_DB_NAME` | Имя БД (default: `sentry`) | ❌ |
| `SENTRY_MAIL_HOST` | SMTP сервер | ❌ |
| `SENTRY_MAIL_PORT` | SMTP порт (default: `587`) | ❌ |
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
```text

### Остановка / запуск

```bash
platform stop sentry
platform start sentry
```text

### Статус

```bash
platform status sentry
docker ps | grep sentry
```text

### Очистка старых событий

Sentry автоматически чистит старые события через `sentry-cron`. Настройка в UI:
- **Settings → Data → Data Scrubbing**
- **Retention**: настройка хранения событий

## 🔍 Интеграция с приложениями

### Python

```bash
pip install sentry-sdk
```text

```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://PUBLIC_KEY@apps.openedu.urfu.ru/sentry/PROJECT_ID",
    traces_sample_rate=1.0,
)
```text

### JavaScript

```bash
npm install @sentry/browser
```text

```javascript
import * as Sentry from "@sentry/browser";

Sentry.init({
  dsn: "https://PUBLIC_KEY@apps.openedu.urfu.ru/sentry/PROJECT_ID",
});
```text

DSN получается в: **Project Settings → Client Keys (DSN)**

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
```text

- **platform_network**: `sentry-web` (для Caddy)
- **sentry_internal**: все сервисы (изолированно)

## 🐛 Troubleshooting

### Сервис не стартует

```bash
# Проверить логи
docker logs sentry-web

# Частые причины:
# 1. SENTRY_SECRET_KEY не задан или короткий
# 2. Нет миграций — выполнить: docker exec -it sentry-web sentry upgrade
# 3. PostgreSQL не готов — проверить: docker logs sentry-postgres
```text

### Health check failed

```bash
# Проверить health endpoint
docker exec -it sentry-web wget -O- http://localhost:9000/api/0/health/

# Если 502 Bad Gateway:
# 1. Проверить, что sentry-web запущен: docker ps
# 2. Проверить логи: docker logs sentry-web
# 3. Возможно долгий старт — подождать 1–2 минуты
```text

### Worker не обрабатывает события

```bash
# Проверить worker
docker logs sentry-worker

# Проверить Redis
docker exec -it sentry-redis redis-cli ping
# Должен вернуть: PONG

# Перезапустить worker
docker restart sentry-worker
```text

### Ошибки БД

```bash
# Проверить PostgreSQL
docker exec -it sentry-postgres pg_isready -U sentry

# Применить миграции
docker exec -it sentry-web sentry upgrade

# Ручной backup БД
docker exec -it sentry-postgres pg_dump -U sentry sentry > sentry_backup.sql
```text

## 📝 Примечания

- Clickhouse **НЕ включён** (режим минимального оверхеда)
- Backup **отключён** (если нужен — см. [`service.local.yml.example`](service.local.yml.example))
- Email уведомления **опциональны** (настраиваются через `SENTRY_MAIL_*`)
- Первый запуск может занять **1–2 минуты** (инициализация БД)

## 📚 Дополнительно

- [Официальная документация Sentry](https://docs.sentry.io/)
- [Self-hosted Guide](https://develop.sentry.dev/self-hosted/)
- [SDK Documentation](https://docs.sentry.io/platforms/)
