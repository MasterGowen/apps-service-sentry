# Обзор Sentry

## Цель

Реализация self-hosted Sentry с минимальной конфигурацией, соответствующей требованиям платформы автодеплоя. Система включает пять контейнеров и не использует Clickhouse.

## Компоненты

### Критические исправления

- Добавление `container_name: sentry-web` в `service.yml`
- Модернизация `docker-compose.yml` до пяти сервисов
- Удаление пустой директории `src/`
- Исправление сетевой конфигурации (использование `platform_network`)

### Внедрённые компоненты

- PostgreSQL 15 — для хранения метаданных и событий
- Redis 7 — для реализации очередей и кэширования
- Sentry Worker — для асинхронной обработки событий
- Sentry Cron — для выполнения периодических задач

### Документация

- Подробная инструкция по развёртыванию
- Примеры конфигурации: `.env.example`, `service.local.yml.example`
- Архитектурные диаграммы

## Характеристики

- Контейнеры: 5 (sentry-web, sentry-worker, sentry-cron, postgres, redis)
- Оперативная память: 1.5–2 ГБ
- Дисковое пространство: 5–10 ГБ
- Сети: platform_network (external), sentry_internal (internal)
- Объёмы: sentry-postgres, sentry-redis
- Clickhouse: не используется
- Резервное копирование: отключено

## Соответствие контракту платформы

- `service.yml` содержит только поддерживаемые поля (meta, routing, health, backup)
- Поле `container_name` присутствует в `routing`
- `docker-compose.yml` использует сеть `platform_network` (external: true)
- Health checks настроены для критичных сервисов
- Отсутствуют не поддерживаемые компоненты (Prometheus, Grafana, Loki, Restic)
