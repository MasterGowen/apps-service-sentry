# Вклад в проект

## Ветвление

```bash
git checkout -b feature/sentry-blueprint-refactor
```

## Коммиты

Выполняются по этапам:

- chore: remove src/, update .gitignore
- fix: add container_name in routing
- feat: implement full Sentry infrastructure
- docs: update DEPLOY.md and DESIGN.md

## Pull Request

```bash
git push origin feature/sentry-blueprint-refactor
```

PR должен содержать:

- Описание изменений
- Проверку валидности YAML
- Подтверждение успешного запуска
