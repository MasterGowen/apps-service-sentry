# Sentry — Система мониторинга ошибок

Self-hosted Sentry для сбора и анализа ошибок в приложениях. Минимальная конфигурация без Clickhouse.

## Описание

Sentry — система мониторинга ошибок с открытым исходным кодом. Поддерживает более 100 языков и фреймворков. Используется для агрегации, анализа и уведомления о сбоях в приложениях.

**Компоненты**:

- **Sentry Web** — пользовательский интерфейс и API (порт 9000)
- **Sentry Worker** — асинхронная обработка событий
- **Sentry Cron** — выполнение периодических задач (очистка, статистика)
- **PostgreSQL 15** — хранение метаданных и событий
- **Redis 7** — реализация очередей и кэширование

**Требования к ресурсам**:

- Оперативная память: 1.5–2 ГБ
- Дисковое пространство: 5–10 ГБ

## Развёртывание

### Подготовка окружения

Скопируйте шаблон переменных окружения:

```bash
cp .env.example .env
```text

Сгенерируйте секретный ключ:

```bash
openssl rand -base64 50
```text

Установите значения в `.env`:

- `SENTRY_SECRET_KEY` — сгенерированный ключ (минимум 50 символов)
- `SENTRY_DB_PASSWORD` — пароль для базы данных

### Развёртывание через платформу

Выполните команду из корня master-сервиса:

```bash
platform deploy sentry
```text

Платформа автоматически:
- Запустит контейнеры
- Настроит маршрутизацию через Caddy
- Активирует health monitoring

### Инициализация базы данных и создание суперпользовател��

После первого запуска выполните миграции:

```bash
docker exec -it sentry-web sentry upgrade --noinput
```text

Создайте учётную запись суперпользователя:

```bash
docker exec -it sentry-web sentry createuser \
  --email admin@example.com \
  --password admin123 \
  --superuser \
  --no-input
```text

### Доступ к интерфейсу

- URL: `https://apps.openedu.urfu.ru/sentry`
- Логин: email из команды `createuser`
- Пароль: пароль из команды `createuser`

После входа необходимо:
- Создать организацию
- Создать проект, выбрав платформу
- Получить DSN в разделе **Project Settings → Client Keys (DSN)**

## Переменные окружения

| Переменная | Описание | Обязательно |
|------------|----------|-------------|
| `SENTRY_SECRET_KEY` | Секретный ключ (минимум 50 символов) | Да |
| `SENTRY_DB_USER` | Имя пользователя PostgreSQL | Да |
| `SENTRY_DB_PASSWORD` | Пароль пользователя PostgreSQL | Да |
| `SENTRY_DB_NAME` | Имя базы данных (по умолчанию: `sentry`) | Нет |
| `SENTRY_MAIL_HOST` | Адрес SMTP-сервера | Нет |
| `SENTRY_MAIL_PORT` | Порт SMTP (по умолчанию: 587) | Нет |
| `SENTRY_MAIL_USERNAME` | Логин SMTP | Нет |
| `SENTRY_MAIL_PASSWORD` | Пароль SMTP | Нет |
| `SENTRY_SERVER_EMAIL` | Email отправителя | Нет |

## Управление

### Логи

Просмотр логов всех сервисов:

```bash
platform logs sentry
```text

Просмотр логов конкретного контейнера:

```bash
docker logs sentry-web -f
docker logs sentry-worker -f
```text

### Остановка и запуск

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

Настройка срока хранения событий осуществляется в интерфейсе:
- **Settings → Data → Data Scrubbing**
- **Retention** — указание периода хранения

## Интеграция с приложениями

### Python

Установка SDK:

```bash
pip install sentry-sdk
```text

Инициализация:

```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://PUBLIC_KEY@apps.openedu.urfu.ru/sentry/PROJECT_ID",
    traces_sample_rate=1.0,
)
```text

### JavaScript

Установка SDK:

```bash
npm install @sentry/browser
```text

Инициализация:

```javascript
import * as Sentry from "@sentry/browser";

Sentry.init({
  dsn: "https://PUBLIC_KEY@apps.openedu.urfu.ru/sentry/PROJECT_ID",
});
```text

DSN доступен в разделе: **Project Settings → Client Keys (DSN)**

## Архитектура

См. архитектурную схему: [ARCHITECTURE.md](ARCHITECTURE.md)
