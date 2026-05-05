# Сводка по рефакторингу Sentry Blueprint

## 🎯 Цель

Трансформировать рахитичный блюпринт с одним контейнером в полноценное self-hosted Sentry решение с минимальным оверхедом, соответствующее контракту платформы автодеплоя.

## 📋 Что будет сделано

### Критические исправления

1. ✅ **service.yml**: добавлен `container_name: sentry-web` в routing (обязательное поле)
2. ✅ **docker-compose.yml**: переход с 1 на 5 контейнеров (web, worker, cron, postgres, redis)
3. ✅ Удаление пустой папки `src/` (используются официальные образы)
4. ✅ Исправление сетевой конфигурации (правильный `platform_network`)

### Новые компоненты

- PostgreSQL 15 (метаданные + события)
- Redis 7 (кэш + очереди)
- Sentry Worker (обработка событий)
- Sentry Cron (фоновые задачи)

### Документация

- Подробный README с инструкциями
- Примеры конфигурации (.env.example, service.local.yml.example)
- Архитектурные диаграммы

## 📊 Характеристики решения

| Параметр | Значение |
|----------|----------|
| Контейнеры | 5 (web, worker, cron, postgres, redis) |
| RAM | ~1.5-2GB |
| Disk | ~5-10GB |
| Сети | 2 (platform_network + sentry_internal) |
| Volumes | 2 (postgres + redis) |
| Clickhouse | ❌ Не включен (минимальный оверхед) |
| Backup | ❌ Отключен (данные не критичны) |

## 🔍 Соответствие контракту платформы

✅ **service.yml** - только поддерживаемые поля (meta, routing, health, backup)  
✅ **routing** - обязательное поле `container_name` присутствует  
✅ **docker-compose** - корректная сеть `platform_network` (external: true)  
✅ **health checks** - настроены для всех критичных сервисов  
✅ **минимализм** - нет Prometheus/Grafana/Loki/Restic  

## 📁 Структура файлов после рефакторинга

```text
sentry/
├── service.yml                    # ✏️ Исправлен (container_name, health endpoint)
├── docker-compose.yml             # ♻️ Полностью переписан (5 сервисов)
├── .env.example                   # ♻️ Расширен (все переменные Sentry)
├── .gitignore                     # ✏️ Дополнен (service.local.yml, volumes/)
├── README.md                      # ♻️ Полностью переписан (подробные инструкции)
├── service.local.yml.example      # ✨ Новый (пример переопределений)
└── plans/                         # ✨ Новая директория
    ├── SUMMARY.md                 # Эта сводка
    ├── sentry-refactoring.md      # Детальный план
    ├── architecture-diagram.md    # Mermaid диаграммы
    └── implementation-spec.md     # Спецификация для кода
```text

**Удалено**: `src/` (не нужна для официальных образов)

## 🚀 Git Workflow

```bash
# Feature branch
git checkout -b feature/sentry-blueprint-refactor

# Коммиты по этапам
1. chore: remove src/, update .gitignore
2. fix: add container_name in routing, update health endpoint  
3. feat: implement full Sentry infrastructure (5 services)
4. feat: add environment variables and local overrides example
5. docs: add detailed README and architecture plans

# Pull Request
git push origin feature/sentry-blueprint-refactor
```text

## ⚠️ Важные замечания

1. **Первый запуск**: требуется инициализация БД через `sentry upgrade`
2. **Суперпользователь**: создается через `sentry createuser` после старта
3. **Старт может занять 1-2 минуты**: из-за миграций PostgreSQL
4. **Email опционален**: можно не настраивать SENTRY_MAIL_* переменные
5. **Не коммитить**: `.env`, `service.local.yml`, `master.db`

## 📚 Созданные документы

1. **[sentry-refactoring.md](sentry-refactoring.md)** - детальный план с архитектурой (8 разделов)
2. **[architecture-diagram.md](architecture-diagram.md)** - Mermaid диаграммы (архитектура, потоки, сети)
3. **[implementation-spec.md](implementation-spec.md)** - спецификация для разработчика (построчно)
4. **[SUMMARY.md](SUMMARY.md)** - краткая сводка (этот файл)

## ✅ Готовность к реализации

План согласован, все детали проработаны. Готово к переключению в **Code mode** для реализации по todo list.

---

**Следующий шаг**: Переключение в Code mode → реализация изменений → git commits → Pull Request
