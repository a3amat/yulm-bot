# 1. High-Level архитектура

Общий обзор системы `yulm-bot`: Telegram-бот как точка входа, ядро бизнес-логики,
внешние интеграции и хранилище данных.

```mermaid
flowchart TB
    subgraph Users["Пользователи"]
        TG["Telegram-клиент"]
    end

    subgraph Bot["yulm-bot"]
        GW["Bot Gateway<br/>(long polling / webhook)"]
        CORE["Core<br/>(бизнес-логика, команды, роутинг)"]
        DB[("Хранилище<br/>(SQLite / PostgreSQL)")]
    end

    subgraph External["Внешние сервисы"]
        LLM["LLM API"]
        EXT["Сторонние API"]
    end

    TG -- "сообщения / команды" --> GW
    GW -- "апдейты" --> CORE
    CORE -- "ответы" --> GW
    CORE <--> DB
    CORE -.-> LLM
    CORE -.-> EXT
```

## Основные слои

1. **Gateway** — приём и отправка сообщений Telegram (aiogram/python-telegram-bot).
2. **Core** — маршрутизация команд, контекст диалога, бизнес-правила.
3. **Storage** — персистентные данные: пользователи, сессии, настройки.
4. **Integrations** — необязательные внешние сервисы (LLM, сторонние API).
