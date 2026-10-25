# 5. Модель данных

ER-диаграмма основной персистентной модели.

```mermaid
erDiagram
    USERS {
        bigint id PK "telegram user id"
        string username
        string first_name
        bool is_active
        datetime created_at
    }

    SESSIONS {
        bigint id PK
        bigint user_id FK
        json state_data "текущее состояние FSM"
        datetime updated_at
    }

    MESSAGES {
        bigint id PK
        bigint user_id FK
        text content
        string direction "in / out"
        datetime ts
    }

    SETTINGS {
        bigint user_id FK "PK"
        json preferences
    }

    USERS ||--o{ SESSIONS : "имеет активные"
    USERS ||--o{ MESSAGES : "переписываетcя"
    USERS ||--|| SETTINGS : "настраивает"
```

## Примечания

1. `SESSIONS.state_data` сериализуется aiogram FSM (SqliteStorage/RedisStorage) либо аналогом.
2. `MESSAGES` нужен для истории диалога и аналитики; при росте — партиционирование по месяцам.
3. Секреты (токен бота, API-ключи) **не хранятся в БД** — только через переменные окружения.
