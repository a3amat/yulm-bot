# 02. Доменная модель и границы контекстов (BCF)

Проектируем «сверху вниз»: сначала границы, потом сервисы. Границы контекстов = будущие границы микросервисов. Это и есть ответ на вопрос «как не пересекаться разным людям»: изоляция закладывается в модели, а не латается на уровне кода.

## Карта bounded contexts

```
┌─────────────────────────────────────────────────────────────┐
│                      Edge / API Gateway                      │
└───┬─────────────┬──────────────┬──────────────┬─────────────┘
    │             │              │              │
┌───▼────┐  ┌─────▼─────┐  ┌─────▼──────┐  ┌────▼─────┐
│Identity│  │ Content   │  │ Publishing │  │ Billing  │
│&Tenancy│  │(материалы,│  │ (jobs,     │  │(планы,   │
│(auth,  │  │ версии,   │  │ расписание,│  │ подписки,│
│workspace│ │ медиа)    │  │ статусы)   │  │ лимиты)  │
│, роли) │  └─────┬─────┘  └─────┬──────┘  └──────────┘
└────────┘        │              │
                  │         ┌────▼──────────────┐
                  │         │ Channel           │
                  └────────►│ Integration Layer │
                            │ (адаптеры площадок│
                            │ + Token Vault)    │
                            └───────────────────┘
        Сквозные: Media Processing, AI Adaptation, Audit/History
```

### 1. Identity & Tenancy (владение: кто и где живёт)
- Сущности: `User`, `Workspace`, `WorkspaceMember(role)`, `ApiKey`.
- Правила: пользователь может состоять в нескольких workspace; все доменные данные привязаны к `workspace_id`; JWT содержит активный workspace; смена workspace = смена области видимости запроса.
- Публичный контракт: `GET /me/workspaces`, события `workspace.created`, `member.role.changed`.

### 2. Content (авторский контент, независимый от площадок)
- Сущности: `Material` (article/video/image/mixed), `MaterialVersion` (append-only!), `MediaAsset` (файл в S3 + вариации), `Tag`.
- Правила: редактирование не мутирует версию — создаёт новую; история материала = список версий; media хранится в объектном хранилище, в БД только ссылки/метаданные.
- Контракт: CRUD материалов, `POST /materials/{id}/versions`, события `material.version.created`.

### 3. Publishing (оркестрация публикации)
- Сущности: `PublicationJob` (M каналов × версия V во время T), `PublicationAttempt` (по одному на канал, state machine), `ScheduleRule`.
- Правила: job → attempts; успех = агрегат статусов attempts (partial_success допустим); идемпотентность через `Idempotency-Key` на attempt; ретраи по backoff; автопауза при отключённом аккаунте.
- Контракт: `POST /publications`, `GET /publications/{job}`, события `publication.attempt.succeeded|failed|retrying`.

### 4. Channel Integration Layer (адаптеры + креды)
- Сущности: `AccountConnection` (workspace ↔ площадка), `ChannelCapability` (что умеет канал), `AdaptedPayload` (готовый к отправке формат).
- **Контракт адаптера (SDK)** — единый интерфейс для всех площадок:
  ```ts
  interface ChannelAdapter {
    id: string;                              // "telegram", "vk"…
    capabilities(): ChannelCapability[];     // text/photo/video/carousel, лимиты
    validate(payload: AdaptedPayload): Result;      // dry-run проверка без отправки
    preview(payload: AdaptedPayload): RenderedPreview;
    publish(conn: Credentials, payload: AdaptedPayload, idemKey: string): PublicationResult;
    verifyWebhook?(req: RawRequest): boolean;       // входящие события площадок
  }
  ```
- Новая соцсеть = новый пакет-адаптер, реализующий интерфейс + манифест capabilities. Ядро Publishing ничего не знает о конкретных площадках — только про контракты. Это ключевая точка расширения.
- **Token Vault**: отдельный сервис/модуль, единственный, кто хранит и расшифровывает OAuth-токены (AES-GCM, master key в KMS/Vault, ротация ключей). Адаптеры получают краткосрочные секреты по `connection_id` + scope. Даже компромисс воркера публикации не утечёт токены всех клиентов.

### 5. Billing (деньги и права)
- Сущности: `Plan` (Starter/Growth/Pro), `Subscription` (month/year, trial), `Entitlement` (лимиты: каналы/публикации/AI-запросы/команда), `UsageCounter`, `Invoice`.
- Правила: биллинг-провайдер (Stripe/Paddle/ЮKassa) — источник истины по платежам; entitlements кешируются локально для быстрой проверки middleware'ом на каждом запросе публикации; soft-limit (предупреждение) vs hard-limit (блок).
- Контракт: checkout session, webhook-обработчик событий провайдера, события `subscription.activated|canceled|limit.exceeded`.

### 6. Сквозные сервисы
- **Media Processing**: ресайз/транскод видео (FFmpeg), генерация превью, форматы под каналы (IG 4:5, TG фото, blog hero). Асинхронные задачи из очереди.
- **AI Adaptation**: конвейер «материал → варианты под канал» (сокращение, переписывание тона, тред-разбиение, alt-текст, хэштеги). Провайдер-абстракция (OpenAI/Anthropic/open-source LLM), cost control, human-in-the-loop (вариант принимает автор).
- **Audit & History**: append-only журнал всех значимых действий (кто что опубликовал/изменил/оплатил) — это и «возможность хранить историю», и база для экспорта данных.

## Как контексты общаются

- Синхронно (запрос-ответ): UI → API Gateway → сервисы (CRUD, проверки entitlements).
- Асинхронно (события через шину): `material.version.created` → AI-адаптер предлагает варианты; `publication.attempt.succeeded` → UsageCounter++, Audit пишет запись; `subscription.canceled` → Publishing ставит jobs на паузу.
- MVP допускает прямые вызовы модулей внутри одного процесса (modular monolith) — но **границы и контракты те же**, поэтому вынос любого контекста в отдельный сервис = замена прямого вызова на RPC/event без переписывания логики.

## Почему именно так (аргументы против альтернатив)

- *Не «сервис на площадку»* (VK-service, TG-service…): слишком рано, N×运维 overhead. Вместо этого один Publishing + библиотека адаптеров; отдельные деплои воркеров возможны позже (например, изолировать тяжёлые видео-загрузки).
- *Не чистый монолит*: задача изначально распределённая (long-running jobs, внешние API с непредсказуемой latencey, разные профили нагрузки) — async-скелет выгоден с первого дня.
- *Модульный монолит как старт* даёт скорость одной команды + дисциплину границ; риск «расползания» снижаем lint-правилами (запрет импортов между модулями кроме контрактов) и contract-тестами.
