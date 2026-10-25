# 04. Каталог сервисов

Формат: ответственность → данные (кем владеет) → API/события → триггер выделения в отдельный микросервис.

## 1. gateway (API Gateway / BFF)
- **Отв**: единая точка входа, аутентификация JWT, маршрутизация по контекстам, rate limiting, агрегация ответов для UI (BFF).
- **Владеет**: нет персистентных данных (кроме refresh-token storage).
- **Выделение**: уже отдельный деплой с начала (тонкий stateless).

## 2. identity-service
- **Отв**: пользователи, workspace'ы, члены и роли, приглашения, API-ключи клиентов.
- **Владеет**: `users`, `workspaces`, `workspace_members`, `api_keys`.
- **API**: register/login/refresh, CRUD workspaces, members/roles.
- **События**: `user.created`, `workspace.created`, `member.role.changed`.
- **Выделение**: остаётся сервисом всегда (общая зависимость всех).

## 3. content-service
- **Отв**: материалы, версии, черновики, теги, media-метаданные; presigned upload.
- **Владеет**: `materials`, `material_versions` (append-only), `media_assets`, `tags`.
- **API**: CRUD materials, versions, diff версий, upload-request/download.
- **События**: `material.version.created`, `media.uploaded`, `material.archived`.
- **Выделение**: при росте медиа — вынести upload/processing отдельно.

## 4. publishing-service (оркестратор)
- **Отв**: создание jobs, state machine attempts, расписание, частичный успех, ретраи, отмена/повтор.
- **Владеет**: `publication_jobs`, `publication_attempts`, `schedule_rules`, `idempotency_keys`.
- **API**: create job, get status, cancel, retry-failed, list history (совместно с audit).
- **События**: исходящие `publication.attempt.*`, `job.completed`; входящие `subscription.*` (пауза при неуплате).
- **Выделение**: Publish Worker выносится сразу во 2-й контейнер (тот же код, роль worker); далее при масштабе — per-channel очереди.

## 5. channel-integration (адаптеры + vault)
- **Отв**: реализация контракта адаптеров площадок, capabilities, dry-run validate/preview; OAuth flow callback; Token Vault.
- **Владеет**: `account_connections` (без секретов — только ссылки на vault), `oauth_states`, секреты в vault-backed хранилище.
- **API**: connect/start, callback, disconnect, test-connection, capabilities, preview.
- **События**: `connection.connected|revoked|token.refresh_failed`.
- **Выделение**: каждый адаптер — npm/pip пакет внутри сервиса; «тяжёлые» адаптеры (YouTube upload) — отдельный воркер позже.

## 6. billing-service
- **Отв**: планы, подписки (month/year/trial), entitlements, usage counters, инвойсы, webhooks провайдера.
- **Владеет**: `plans`, `subscriptions`, `entitlements`, `usage_counters`, `invoices`.
- **API**: catalog, checkout-session, portal link, my-subscription, limits check (для middleware).
- **События**: `subscription.activated|renewed|canceled|past_due`, `limit.exceeded`.
- **Выделение**: отдельный сервис с первого дня из-за комплаенса (PCI-скоуп изолирован).

## 7. ai-adaptation-service
- **Отв**: конвейер адаптации контента под канал: промпт-шаблоны по capabilities, варианты, стоимость запросов, кеш.
- **Владеет**: `adaptation_runs`, `prompt_templates`, `llm_usage`.
- **API**: adapt(material_version, channels[]) → варианты; accept/reject варианта.
- **События**: `adaptation.completed`.
- **Выделение**: отдельный воркер с первого дня (внешние LLM = непредсказуемая latency, нужен budget guard).

## 8. media-processor (воркер)
- **Отв**: транскод видео, ресайз изображений под форматы каналов, превью, водяные знаки.
- **Владеет**: производными артефактами в S3 (`variants`).
- **События**: входит `media.uploaded`, исходит `media.variant.ready`.

## 9. scheduler (воркер)
- **Отв**: скан `schedule_rules` due → эмит publication jobs; таймеры ретраев.
- Простой single-thread, leader-elected (Redlock) — дублы исключены.

## 10. audit-history-service
- **Отв**: append-only журнал значимых событий (публикации, изменения материалов, платежи, доступы к секретам), экспорт данных tenant'а, retention.
- **Владеет**: `audit_events` (партиционирование по месяцам).
- **API**: search/filter по workspace, export CSV/JSON.
- Только запись через события — ни один сервис не может переписать историю.

## Матрица зависимостей (кто кого зовёт)

| Сервис | Синхронные вызовы | Подписка на события |
|---|---|---|
| gateway | все core-сервисы | — |
| publishing | billing.check-entitlement, integration.validate | subscription.*, schedule due |
| integration | vault | connection.* |
| billing | proвайдер платежей | publication.attempt.succeeded (usage++) |
| ai-adaptation | LLM providers | material.version.created |
| audit | — | почти все доменные события |

Правило дисциплины: **запрещены** прямые вызовы «через слой» (например, AI-воркер не дергает Telegram напрямую — только через Publishing→Integration). Линтер архитектуры (dependency-cruiler/eslint boundaries) закрепляет это в CI.
