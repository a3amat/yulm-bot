# 2. Диаграмма компонентов

Внутреннее устройство бота: модули и зависимости между ними.

```mermaid
flowchart LR
    subgraph Entry["Точка входа"]
        MAIN["main.py<br/>(запуск, конфиг)"]
    end

    subgraph Handlers["Слой обработчиков"]
        CMD["Command Handlers<br/>/start, /help, ..."]
        MSG["Message Handlers"]
        CB["Callback Handlers<br/>(инлайн-кнопки)"]
    end

    subgraph Services["Сервисный слой"]
        ROUTER["Dialog Router<br/>(FSM / состояния)"]
        BIZ["Business Services"]
        INTEG["Integrations Client"]
    end

    subgraph Infra["Инфраструктура"]
        REPO["Repositories<br/>(DAL)"]
        CFG["Config / Env"]
        LOG["Logging / Metrics"]
    end

    DB[("Database")]
    TGAPI["Telegram Bot API"]

    MAIN --> Handlers
    MAIN --> Services
    MAIN --> Infra
    Handlers --> Services
    CMD --> ROUTER
    MSG --> ROUTER
    CB --> BIZ
    ROUTER --> BIZ
    BIZ --> REPO
    BIZ --> INTEG
    INTEG --> TGAPI
    REPO --> DB
    Handlers --> TGAPI
    Services --> LOG
    Infra --> CFG
```

## Назначение компонентов

| Компонент | Ответственность |
|---|---|
| `main` | инициализация, запуск polling/webhook |
| `Handlers` | тонкий слой: валидация входа, вызов сервисов |
| `Dialog Router` | машина состояний диалога (FSM) |
| `Business Services` | правила, сценарии, интеграционная логика |
| `Repositories` | доступ к данным (Data Access Layer) |
| `Integrations Client` | обёртки над внешними API |
| `Config` | переменные окружения, секреты |
