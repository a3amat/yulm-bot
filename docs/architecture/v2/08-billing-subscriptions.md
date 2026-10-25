# 08. Биллинг и подписки

## Модель монетизации (предложение)

| | Trial | Starter | Growth | Pro | Agency |
|---|---|---|---|---|---|
| Цена мес/год (год −2 мес) | 14 дней | $19 / $190 | $49 / $490 | $99 / $990 | от $249 |
| Workspace'ы | 1 | 1 | 3 | 10 | безлимит |
| Подключённые каналы | 2 | 5 | 20 | 60 | безлимит |
| Публикаций/мес | 30 | 200 | 2000 | ∞ (fair use) | ∞ |
| AI-адаптации/мес | 20 | 100 | 1000 | безлимит* | безлимит* |
| Команда | 1 | 3 | 10 | 30 | ∞ |
| Медиа-хранилище | 1 ГБ | 5 ГБ | 50 ГБ | 200 ГБ | +тарифный объём |
| White-label/API/webhooks | — | — | API | API+webhooks | всё |

\* AI с soft-cap по себестоимости LLM-токенов, чтобы Pro-клиент не сжигал маржу.

## Техническая схема

```
UI → billing-service → [Stripe Billing API]   (Paddle = merchant of record: сам платит налоги/VAT — проще для глобального рынка; Stripe = гибче и дешевле при своём compliance)
     ▲ webhook: invoice.paid, customer.subscription.updated/deleted, payment_failed
     │
entitlements cache (Redis) ← middleware на каждом write-запросе publishing/content
usage_counters ← событие publication.attempt.succeeded (+ ai.completed, media.uploaded)
```

### Принципы
1. **Не изобретаем биллинг**: инвойсы, налоги, продления, dunning (цепочка напоминаний при неудачном списании), промокоды, портал управления подпиской — всё отдаём провайдеру (Stripe Billing Portal / Paddle). Свой код хранит только маппинг `workspace ↔ subscription ↔ entitlements`.
2. **Entitlements как декларация** (`jsonb` у плана): код нигде не знает слово «Growth», только `max_channels`, `posts_per_month`, `team_seats`. Новый тариф = запись в БД, не релиз.
3. **Проверка лимитов**: синхронно перед действием (cheap Redis counter + cache плана); мягкие лимиты предупреждают, жёсткие блокируют с кодом ошибки `UPGRADE_REQUIRED` и ссылкой на checkout.
4. **Webhook'и идемпотентны** (store event id), обрабатываются очередью, ретраятся; состояние подписки в нашей БД — кэш состояния Stripe, источник истины — провайдер. Сверка раз в сутки reconciliation job'ом.
5. **Жизненный цикл**: trial → active → past_due (grace 7д, публикации приостановлены, черновики доступны) → canceled (данные храним N дней для реактивации, затем GDPR-удаление). Годовой план — скидка за предоплату, возврат пропорционально (политика).
6. **Метрики для владельца продукта**: MRR/ARR по планам, churn, expansion (апгрейды), usage per plan — отдельная read-only витрина (materialized views или метрический warehouse позже).

### Flow покупки
```
выбор плана+interval → POST /billing/checkout {plan, interval} → Stripe Checkout Session
→ пользователь платит → webhook checkout.session.completed → subscription.active
→ события в шину → Publishing снимает паузу jobs workspace'а → UI: «Подписка активна»
```

### Особые случаи
- Смена плана mid-period: upgrade — немедленный prorated charge; downgrade — применяется со следующего периода.
- Расторжение по инициативе клиента: `cancel_at_period_end=true`, доступ до конца оплаченного периода.
- Неудачное списание recurring: dunning от провайдера + наш grace + авто-пауза публикаций за 3 дня до end-of-grace (уведомлениями).
- Рынок РФ: ЮKassa/CloudPayments + собственные счета? Тогда отказ от «готового» portal, свой модуль продлений — оцениваем трудозатраты ×2..3; альтернатива — Paddle работает с рядом стран СНГ через локальные методы оплаты, проверяем актуальный список.

## Что НЕ делаем на MVP
- Свои платежи/PCI, сложные промо-движки, reseller-модель агентств (сначала прямая подписка), usage-based биллинг поверх flat-rate (добавим как опцию «пакеты публикаций» позже).
