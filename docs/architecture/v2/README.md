# CrossPulse — архитектура платформы мультипостинга (v2)

> Рабочее название: **CrossPulse** (можно заменить). Это переработка архитектуры под реальную задачу проекта:
> загрузка контента (статьи, видео, изображения/картины) одновременно в разные соцсети, блоги и мессенджеры —
> для множества независимых пользователей (workspace'ов), с биллингом, подписками, историей и AI-адаптацией контента.

## Документы

| Документ | О чём |
|---|---|
| [01-product-and-requirements.md](01-product-and-requirements.md) | Что строим: сценарии, функциональные/нефункциональные требования, доменные понятия |
| [02-domain-model.md](02-domain-model.md) | Границы контекстов (DDD/BCF): Content, Publishing, Integrations, Identity, Billing |
| [03-high-level-architecture.md](03-high-level-architecture.md) | High-Level схема микросервисов + выбор сообщений/шин + API-first |
| [04-services-catalog.md](04-services-catalog.md) | Каталог сервисов: ответственность, данные, события каждого сервиса |
| [05-data-model.md](05-data-model.md) | Схема данных: workspace-изоляция, токены, версии материалов, публикаций, биллинг |
| [06-publishing-flow.md](06-publishing-flow.md) | Flow публикации: state machine, retry, частичный успех, dry-run превью |
| [07-security-isolation.md](07-security-isolation.md) | Изоляция клиентов, шифрование OAuth-токенов, RBAC, audit log |
| [08-billing-subscriptions.md](08-billing-subscriptions.md) | Подписки (месяц/год), лимиты, интеграция Stripe/Paddle, webhook'и |
| [09-ai-adaptation.md](09-ai-adaptation.md) | AI-конвейер адаптации контента под форматы площадок |
| [10-mvp-roadmap.md](10-mvp-roadmap.md) | Поэтапный план: от одного канала (Telegram) до матрицы площадок |
| diagrams/ | Mermaid/PlantUML схемы (high-level, sequence, deployment, ER) |

## TL;DR — ключевые решения

1. **Модель зрелости «монолит → микросервисы»**: начинаем с модульного ядра (modular monolith) с чёткими границами контекстов; микросервисы вырезаем только когда это оправдано нагрузкой/командой. Границы сервисов спроектированы заранее — миграция не потребует переписывания.
2. **Плагин-адаптеры площадок** — отдельный слой `Channel Adapters` с единым контрактом (`publish`, `validate`, `preview`, `capabilities`). Новая соцсеть = новый адаптер, ядро не трогаем. Первый адаптер — Telegram Bot API (самый простой, без ревью доступа).
3. **Workspace-изоляция на уровне запросов**: каждый клиент со своим набором аккаунтов; все таблицы с `workspace_id`, RLS в PostgreSQL, OAuth-токены в отдельном зашифрованном хранилище vault-сервиса.
4. **Pipeline as a Service**: контент проходит цепочку *источник → нормализация (Markdown) → версия → AI-адаптация под канал → превью → публикация*. Публикации идемпотентны, с state machine и retry — частичный успех (опубликовано в 3 из 5 сетей) это штатный сценарий, а не ошибка.
5. **Async-first**: очередь задач (Redis Streams / RabbitMQ / NATS — на MVP хватает Redis), события доменной шины, планировщик отложенных постов.
6. **Биллинг через провайдера** (Stripe или Paddle/ЮKassa по рынку): подписки month/year, entitlements (что доступно по тарифу), лимиты публикаций/каналов/AI-запросов; история — неизменяемый журнал (append-only) публикаций и версий материалов.

## С чего начать (MVP за ~3 недели)

```
Неделя 1: ядро + Telegram
  - монorepo, Docker Compose (Postgres, Redis, MinIO)
  - Auth (email+OAuth провайдеры) + Workspace + CRUD материалов (md/wysiwyg)
  - Adapter SDK + TelegramAdapter (text/photo/video) + публикация вручную "опубликовать сейчас"
Неделя 2: pipeline и история
  - versions/drafts, publish jobs + retry, journal публикаций (who/what/where/when/status)
  - нормализатор Markdown -> форматы каналов (HTML для blog, plain для TG)
Неделя 3: биллинг-скелет + планировщик
  - Stripe subscriptions (test mode), entitlements middleware, лимиты
  - cron/scheduler отложенных постов, dry-run превью в UI
Далее инкрементами: VK -> Twitter/X -> Instagram -> YouTube -> блог-платформы, AI-адаптация, webhooks-импорт.
```

Подробнее — в [10-mvp-roadmap.md](10-mvp-roadmap.md).
