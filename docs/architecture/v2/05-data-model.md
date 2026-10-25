# 05. Модель данных

PostgreSQL. Все доменные таблицы содержат `workspace_id` + включённый RLS (см. [07-security-isolation.md](07-security-isolation.md)). Ниже — ядро схемы; полный ER в `diagrams/er.mmd`.

## Конвенции
- PK: `uuid` (v7 — сортируется по времени). FK индексируются всегда. Мягкое удаление только там, где нужна история (`deleted_at`).
- Время: `timestamptz`, UTC. Append-only таблицы не UPDATE'ятся вообще (versions, audit, attempts после terminal-статуса).

## Identity & Tenancy

```sql
CREATE TABLE users (
  id uuid PRIMARY KEY,
  email citext UNIQUE NOT NULL,
  password_hash text,              -- nullable при OAuth-login
  name text, avatar_url text,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE workspaces (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  slug citext UNIQUE NOT NULL,     -- публичный для URL
  owner_id uuid REFERENCES users(id),
  plan_id uuid,                    -- кэш активного плана (истина в billing)
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
  workspace_id uuid REFERENCES workspaces(id) ON DELETE CASCADE,
  user_id uuid REFERENCES users(id),
  role text CHECK (role IN ('owner','admin','editor','publisher','viewer')),
  invited_by uuid, joined_at timestamptz DEFAULT now(),
  PRIMARY KEY (workspace_id, user_id)
);
```

## Content

```sql
CREATE TABLE materials (
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL REFERENCES workspaces(id),
  kind text CHECK (kind IN ('article','video','image','mixed')),
  title text NOT NULL,
  current_version_id uuid,         -- указатель на последнюю версию
  status text CHECK (status IN ('draft','ready','archived')) DEFAULT 'draft',
  tags text[],
  created_at timestamptz DEFAULT now(),
  deleted_at timestamptz
);

CREATE TABLE material_versions (           -- append-only = история редактирования
  id uuid PRIMARY KEY,
  material_id uuid NOT NULL REFERENCES materials(id),
  version_no int NOT NULL,                 -- 1,2,3…
  body_md text NOT NULL,                   -- нормализованный Markdown как source of truth
  frontmatter jsonb,                       -- title, description, cover…
  author_id uuid, created_at timestamptz DEFAULT now(),
  UNIQUE (material_id, version_no)
);

CREATE TABLE media_assets (
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL,
  material_version_id uuid REFERENCES material_versions(id),
  kind text CHECK (kind IN ('image','video','file')),
  storage_key text NOT NULL,               -- s3://bucket/ws/{id}/…
  mime text, size_bytes bigint, width int, height int, duration_s numeric,
  checksum text,                           -- dedup + integrity
  variants jsonb,                          -- {"tg_1280": key, "ig_4x5": key, …}
  created_at timestamptz DEFAULT now()
);
```

## Publishing

```sql
CREATE TABLE account_connections (          -- в channel-integration, но tenant-scoped
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL,
  channel text NOT NULL,                    -- 'telegram','vk','youtube','medium'…
  external_account_id text,                 -- chat_id / group_id / channel handle
  display_name text,
  status text CHECK (status IN ('pending_oauth','connected','reauth_required','revoked')),
  secret_ref text NOT NULL,                 -- ссылка на Token Vault, НЕ сам токен
  capabilities jsonb,                       -- снимок возможностей канала
  connected_by uuid, created_at timestamptz DEFAULT now(),
  UNIQUE (workspace_id, channel, external_account_id)
);

CREATE TABLE publication_jobs (
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL,
  material_version_id uuid NOT NULL,
  requested_by uuid NOT NULL,
  scheduled_for timestamptz,                -- NULL = немедленно
  status text CHECK (status IN ('queued','running','partial_success','success','failed','canceled')),
  created_at timestamptz DEFAULT now()
);

CREATE TABLE publication_attempts (         -- один канал внутри jobа; история = это
  id uuid PRIMARY KEY,
  job_id uuid NOT NULL REFERENCES publication_jobs(id),
  workspace_id uuid NOT NULL,
  connection_id uuid NOT NULL,
  idempotency_key text UNIQUE NOT NULL,     -- hash(job_id, connection_id)
  state text CHECK (state IN
    ('pending','validating','sending','retry_wait','succeeded','failed','skipped')),
  attempt_no int DEFAULT 1,
  adapted_payload jsonb,                    -- что реально отправили (для аудита/повтора)
  remote_post_url text,                     -- ссылка на пост после успеха
  error_code text, error_message text,
  next_retry_at timestamptz,
  started_at timestamptz, finished_at timestamptz
);
```

## Billing

```sql
CREATE TABLE plans (
  id uuid PRIMARY KEY, code text UNIQUE,       -- starter/growth/pro
  price_month_cents int, price_year_cents int, currency char(3),
  entitlements jsonb NOT NULL,   -- {"max_channels":3,"max_posts_per_month":100,"ai_tokens":50k,"team_seats":1}
  active bool DEFAULT true
);

CREATE TABLE subscriptions (
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL,
  plan_id uuid NOT NULL REFERENCES plans(id),
  interval text CHECK (interval IN ('month','year')),
  status text CHECK (status IN ('trialing','active','past_due','canceled')),
  provider text, provider_sub_id text UNIQUE, -- Stripe sub id — источник истины
  current_period_end timestamptz NOT NULL,
  cancel_at_period_end bool DEFAULT false
);

CREATE TABLE usage_counters (
  workspace_id uuid NOT NULL, period date NOT NULL,   -- день/месяц
  metric text NOT NULL,                               -- posts_published, ai_requests, uploads_gb
  value numeric DEFAULT 0,
  PRIMARY KEY (workspace_id, period, metric)
);
```

## Audit / History (append-only, партиционирование по месяцам)

```sql
CREATE TABLE audit_events (
  id bigint GENERATED ALWAYS AS IDENTITY,
  workspace_id uuid NOT NULL,
  actor_id uuid,
  event_type text NOT NULL,        -- publication.succeeded, material.version.created, payment.captured…
  entity text, entity_id text,
  payload jsonb,
  occurred_at timestamptz DEFAULT now()
) PARTITION BY RANGE (occurred_at);
```

## Изоляция и целостность
- **RLS**: политика на каждую tenant-таблицу: `USING (workspace_id = current_setting('app.workspace_id')::uuid)`; роль приложения без обхода — только у миграций/админки.
- **Секреты**: нигде в БД нет токенов — только `secret_ref` в Vault. Компрометация дампа БД ≠ компрометация аккаунтов клиентов.
- **История**: версии материалов + attempts + audit образуют полную картину «что публиковалось когда и откуда»; повтор публикации = новый job со ссылкой на старую версию.
- **Удаление tenant'а** (GDPR): каскад по `workspace_id` + удаление объектов S3 + реестр vault; audit помечается анонимизацией, метрики платежей остаются по фискальным требованиям.
