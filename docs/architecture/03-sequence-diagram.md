# 3. Sequence-диаграмма: обработка сообщения

Типовой сценарий: пользователь отправляет сообщение боту.

```mermaid
sequenceDiagram
    autonumber
    actor U as Пользователь (Telegram)
    participant T as Telegram Bot API
    participant G as Bot Gateway
    participant R as Dialog Router (FSM)
    participant S as Business Service
    participant D as Database

    U->>T: Отправляет сообщение / команду
    T->>G: Update (webhook / long polling)
    G->>R: Маршрутизировать апдейт
    R->>D: Загрузить состояние диалога
    D-->>R: Состояние + контекст
    R->>S: Вызвать обработчик сценария
    S->>D: Сохранить данные / новый state
    S-->>G: Ответ (текст / кнопки / медиа)
    G->>T: sendMessage / editMessage
    T-->>U: Бот отвечает
```

## Варианты сбоев

- **Timeout внешнего API** — повтор с экспоненциальной задержкой, деградация до ответа «попробуйте позже».
- **Ошибка записи в БД** — транзакция откатывается, пользователю возвращается нейтральное сообщение, ошибка логируется.
