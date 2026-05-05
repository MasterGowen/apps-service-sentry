# Архитектура Sentry

## Архитектура компонентов

Sentry состоит из пяти сервисов:

- sentry-web — пользовательский интерфейс и API (порт 9000)
- sentry-worker — обработка очереди событий
- sentry-cron — выполнение периодических задач
- postgres — хранение метаданных и событий
- redis — реализация очередей и кэширование

Сеть platform_network используется только для маршрутизации через Caddy.

```mermaid
graph TB
    subgraph External["Внешний мир"]
        User[Пользователь]
    end
    
    subgraph Platform["Platform Network - external"]
        Caddy[Caddy Proxy<br/>apps.openedu.urfu.ru/sentry]
    end
    
    subgraph SentryServices["Sentry Services"]
        Web[sentry-web<br/>:9000<br/>UI + API]
        Worker[sentry-worker<br/>event processing]
        Cron[sentry-cron<br/>cleanup tasks]
    end
    
    subgraph Infrastructure["Infrastructure - sentry_internal"]
        Postgres[(PostgreSQL 15<br/>metadata + events)]
        Redis[(Redis 7<br/>cache + queues)]
    end
    
    User -->|HTTPS| Caddy
    Caddy -->|HTTP| Web
    
    Web -->|read/write| Postgres
    Web -->|pub/sub| Redis
    
    Worker -->|read/write| Postgres
    Worker -->|consume| Redis
    
    Cron -->|cleanup| Postgres
    Cron -->|cache| Redis
    
    style User fill:#e1f5ff
    style Caddy fill:#ffe1e1
    style Web fill:#d4edda
    style Worker fill:#d4edda
    style Cron fill:#d4edda
    style Postgres fill:#fff3cd
    style Redis fill:#fff3cd
```

## Потоки данных

```mermaid
sequenceDiagram
    participant App as Client App
    participant Web as sentry-web
    participant Redis as Redis Queue
    participant Worker as sentry-worker
    participant DB as PostgreSQL
    
    App->>Web: POST /api/store/ (error event)
    Web->>Redis: Enqueue event
    Web-->>App: 200 OK (async)
    
    Redis->>Worker: Consume event
    Worker->>DB: Store processed event
    Worker->>DB: Update statistics
    
    Note over Web,DB: Асинхронная обработка<br/>для быстрого ответа клиенту
```

## Состав контейнеров

```mermaid
graph LR
    subgraph Sentry["docker-compose.yml - 5 сервисов"]
        W[sentry-web<br/>image: sentry:24<br/>port: 9000]
        WK[sentry-worker<br/>image: sentry:24<br/>no ports]
        CR[sentry-cron<br/>image: sentry:24<br/>no ports]
        PG[postgres<br/>image: postgres:15-alpine<br/>internal: 5432]
        RD[redis<br/>image: redis:7-alpine<br/>internal: 6379]
    end
    
    W -.depends_on.-> PG
    W -.depends_on.-> RD
    WK -.depends_on.-> PG
    WK -.depends_on.-> RD
    CR -.depends_on.-> PG
    CR -.depends_on.-> RD
    
    style W fill:#90EE90
    style WK fill:#90EE90
    style CR fill:#90EE90
    style PG fill:#FFD700
    style RD fill:#FFD700
```

## Сетевая топология

```mermaid
graph TB
    subgraph platform_network["platform_network - external: true"]
        Caddy[Caddy]
        SentryWeb[sentry-web<br/>ONLY]
    end
    
    subgraph sentry_internal["sentry_internal - internal"]
        Web2[sentry-web]
        Worker[sentry-worker]
        Cron[sentry-cron]
        PG[postgres]
        Redis[redis]
    end
    
    Caddy -->|proxy| SentryWeb
    
    Web2 --> PG
    Web2 --> Redis
    Worker --> PG
    Worker --> Redis
    Cron --> PG
    Cron --> Redis
```

## Volumes и персистентность

```mermaid
graph LR
    subgraph Volumes["Named Volumes"]
        PGV[sentry-postgres<br/>PostgreSQL data]
        RDV[sentry-redis<br/>Redis data]
    end
    
    subgraph Containers["Контейнеры"]
        PG[postgres]
        RD[redis]
    end
    
    PGV -.mount.-> PG
    RDV -.mount.-> RD
```

## Health Checks

```mermaid
stateDiagram-v2
    [*] --> Starting
    Starting --> Healthy: health check OK
    Starting --> Unhealthy: timeout/fail
    Healthy --> Unhealthy: health check fail
    Unhealthy --> Healthy: health check OK
    Unhealthy --> [*]: max retries
    
    note right of Healthy
        sentry-web: GET /api/0/health/
        postgres:   pg_isready -U user
        redis:      redis-cli ping
    end note
```

**Ресурсы**: ~1.5–2 ГБ ОЗУ, ~5–10 ГБ дискового пространства
**Контейнеры**: 5 (sentry-web, sentry-worker, sentry-cron, postgres, redis)
**Сети**: platform_network (external), sentry_internal (internal)
**Volumes**: sentry-postgres, sentry-redis
