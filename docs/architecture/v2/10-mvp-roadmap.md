# 10. MVP Roadmap — от одного канала к матрице

Принцип: **каждый инкремент — законченный продукт**, которым можно заплатить пользователю. Сначала сквозная тонкая дорожка (thin slice) через всю архитектуру, потом утолщение.

## M0 — Скелет (2–3 дня)
- Монорепо (`apps/api`, `apps/web`, `packages/adapters/*`), Docker Compose: Postgres+RLS миграции, Redis, MinIO.
- CI: lint (границы модулей!), test, build. Структура папок = границы контекстов из [02](02-domain-model.md).
- Choice lock-in: язык (предложение TypeScript/NestJS + React).

## M1 — «Пост в Telegram» (неделя 1) ⭐ ключной milestone
Сквозной thin slice: регистрация → workspace → подключить TG-канал (токен бота / BotFather flow) → написать заметку → опубликовать → увидеть в журнале.
- identity: users/workspaces/members, JWT + RLS middleware.
- content: material v1 (текст + картинка), версии пока одна строка (append-only таблица уже с начала!).
- integration: Adapter SDK + **TelegramAdapter** (sendMessage/sendPhoto), validate/preview/capabilities.
- publishing: синхронная публикация одной кнопки (job/attempt таблицы уже есть; воркер тот же процесс).
- web: онбординг, редактор TipTap, список подключений, журнал.
Критерий готовности: два разных workspace публикуют в свои каналы и не видят данные друг друга (cross-tenant тест зелёный).

## M2 — Асинхронность и история (неделя 2)
- Publish Worker отдельным контейнером BullMQ; retry/backoff/state machine по [06](06-publishing-flow.md); SSE статусов.
- Расписание: ScheduleRule + scheduler (tz пользователя).
- Журнал: attempts с remote_post_url, diff версий материала в UI.
- Media: presigned upload в MinIO, базовый ресайз (sharp) под TG.

## M3 — Биллинг-скелет (неделя 3)
- Stripe test mode: checkout month/year, webhook'и, entitlements cache, лимит каналов/публикаций, upgrade-ошибки в UI.
- Trial 14 дней. Деплой staging + публичный лендинг с waitlist.

→ **M1–M3 = продаваемый MVP.** Дальше растим матрицу площадок и ценность.

## M4 — Вторая площадка: VK (или X — по ответу на вопрос №2)
Главная проверка гипотезы «адаптеры расширяются без правки ядра»: только новый пакет адаптера + OAuth flow + capabilities + превью. Если пришлось менять Publishing — рефакторим контракт (это сигнал, ловим рано).

## M5 — AI-адаптация v1 ([09](09-ai-adaptation.md))
Операторы fit_length + reformat + hashtags для текстовых постов; structured output; diff-view принятия. Считаем стоимость на публикацию, включаем в тарифы.

## M6 — Медиа-конвейер
Видео: загрузка → транскод (ffmpeg worker) → публикации как video-post (TG/VK) или ссылка (YouTube adapter). Картины: галерея/carousel (IG через Meta Graph API, требует app review — стартуем заранее).

## M7 — Команды и роли + блог-площадки
Многопользовательские workspace (editor/publisher/approval-flow «согласование перед публикацией»), WordPress/Medium adapters (HTML-экспорт из Markdown).

## M8 — Рост микросервисности (по факту нагрузки)
Вынос: AI worker → отдельный деплой (есть), media processor → очереди GPU/CPU, integration per-channel rate limiter → edge Redis. Только при метриках: p95 publish latency > SLA, contention очередей, ≥2 команды разработки.

## M9 — Платформенные фичи
Public API + webhooks подписки на события, импорт RSS, аналитика (метрики постов через APIs), white-label для Agency тарифа.

## Метрики успеха каждого этапа
- M1: первый внешний пользователь опубликовал без нашей помощи.
- M3: первая платная подписка (даже символическая).
- M4: подключение новой площадки ≤ 3 рабочих дней кодом.
- M5: ≥30% публикаций используют AI-вариант.
- Постоянно: publish success rate ≥98%, time-to-first-post < 10 мин.

## Риски плана
| Риск | Митигация |
|---|---|
| Доступ к API площадок (app review IG/X занимает недели) | заявки сразу на старте M4, песочницы, приоритет площадок с открытым доступом (TG/VK/WP) |
| Изменение политик автопостинга соцсетей | адаптеры изолируют изменения; fallback-режим «подготовить пост → буфер обмена/API автора» |
| Раздувание MVP до релиза | критерии готовности этапов фиксированы; всё «хотелось бы» — в backlog после M3 |
