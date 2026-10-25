# 4. Схема развёртывания (Deployment)

```mermaid
flowchart TB
    subgraph Cloud["Telegram Cloud"]
        TGBOT["Telegram Bot API"]
    end

    subgraph Host["Сервер / VPS (или PaaS)"]
        subgraph Docker["Docker Compose"]
            APP["yulm-bot<br/>(container)"]
            REDIS[("Redis<br/>(FSM state, кэш)")]
            PG[("PostgreSQL<br/>(данные)")]
        end
        REV["Caddy / Nginx<br/>(TLS-терминация, webhook)"]
    end

    CI["CI/CD<br/>(GitHub Actions)"]

    TGBOT -- "webhook (HTTPS)" --> REV --> APP
    APP <--> REDIS
    APP <--> PG
    CI -- "build & push image" --> Docker
    APP -. "outbound: polling / API calls" .-> TGBOT
```

## Режимы запуска

| Режим | Когда использовать |
|---|---|
| **Long polling** | разработка, нет публичного HTTPS |
| **Webhook + reverse proxy** | прод, выше производительность |

## Компоненты инфраструктуры

- **PostgreSQL** — основные данные; для MVP допустима замена на SQLite.
- **Redis** — состояния FSM и кэш; опционально (можно memory storage в dev).
- **Caddy/Nginx** — TLS для webhook.
- **GitHub Actions** — линты, тесты, сборка образа, деплой.
