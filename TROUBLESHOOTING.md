# Troubleshooting

## Сервис не стартует

Проверьте логи:

```bash
docker logs sentry-web
```text

Возможные причины:

- SENTRY_SECRET_KEY не задан или слишком короткий
- Не выполнены миграции
- PostgreSQL не готов к подключению

## Health check failed

Проверьте доступность health endpoint:

```bash
docker exec -it sentry-web wget -O- http://localhost:9000/api/0/health/
```text

Если ответ 502 — возможно, сервис ещё не запустился (ожидание до 2 минут).

## Worker не обрабатывает события

Проверьте:

- Логи worker: `docker logs sentry-worker`
- Состояние Redis: `docker exec -it sentry-redis redis-cli ping`
