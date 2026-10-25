# 07. Безопасность и изоляция клиентов

## Угрозы, которые закрываем
| Угроза | Мера |
|---|---|
| Утечка данных одного клиента другому (баг в запросе) | PostgreSQL RLS + обязательный `workspace_id` фильтр + интеграционные тесты «cross-tenant» |
| Компрометация токенов площадок (дамп БД/воркера) | Token Vault: шифрование AES-256-GCM, master key в KMS/HSM, секреты не ложатся в логи (`pino-redact`), scoped ephemeral access |
| Перебор чужих ресурсов по ID (IDOR) | Проверка ownership на каждом endpoint; UUID v7 вместо последовательных ID |
| Злоупотребление публичным API | Rate limits per-workspace, API keys с scope, подписанные webhooks (HMAC) |
| Инъекция через контент (XSS в превью, markdown-атаки) | Санитайз HTML (DOMPurify/marketplace-lib), CSP, превью в изолированном iframe/sandbox |
| Компрометация зависимостей адаптеров | SBOM, dependabot, network egress allowlist для воркеров (только домены площадок) |
| Инсайдера-администратора | Audit всех админ-действий, break-glass процедура, least privilege в k8s/облаке |

## Модель доступа

```
User ──(member of)──> Workspace(role) ──owns──> Connections, Materials, Publications, Subscription

Роли: owner > admin (управление командой/биллинг) > publisher (публикует) > editor (редактирует) > viewer (только просмотр).
Матрица permission'ов — декларативная таблица в коде identity-service, тестируется.
```

### JWT claims
```json
{ "sub": "user-uuid", "sid": "session-uuid", "ws": "active-workspace-uuid",
  "role": "editor", "exp": ... }
```
- Access short-lived (10–15 мин), refresh с ротацией и детекцией перевыпуска украденного refresh.
- Смена workspace меняет `ws` claim → все запросы дальше автоматически в новой области; middleware сверяет `ws` из токена с запрашиваемым ресурсом **до** обращения к БД.

## Row-Level Security (пример)

```sql
ALTER TABLE materials ENABLE ROW LEVEL SECURITY;
CREATE POLICY ws_isolation ON materials
  USING (workspace_id = current_setting('app.workspace_id')::uuid);
-- роль приложения НЕ имеет BYPASSRLS; миграции под отдельной ролью
```

Каждое подключение пула после `SET app.workspace_id = $1`. Даже если разработчик забудет WHERE — БД сама не отдаст чужие строки. Это второй эшелон защиты после middleware.

## Token Vault — детали
- Хранение: envelope encryption (DEK на секрет, KEK в KMS/Vault Transit engine). Запись в vault = `secret_ref`; расшифровка только внутри vault-процесса по mTLS + scope (`publish`, `reauth`).
- OAuth flow: state+PKCE для всех провайдеров; callback в integration-service; токен сразу уходит в vault, в HTTP-сессии не задерживается.
- Ротация: фоновый job обновляет refresh-токены до истечения (VK/TG expire-политики разные), при неудаче — статус `reauth_required` + уведомление владельца workspace.
- Аудит: каждый доступ к секрету пишется в audit_events (кто, какой connection, зачем).

## Секреты платформы
- Конфиги через env + sealed-secrets/SOPS; никаких ключей в репозитории (gitleaks в CI).
- Ключи приложений площадок (наш OAuth client) — тоже в vault, отдельно от клиентских токенов.

## Данные пользователей
- Минимизация: храним только необходимое; email-подтверждение; 2FA для owner/admin (TOTP/WebAuthn).
- GDPR/152-ФЗ: экспорт данных workspace (JSON+медиа zip) self-service; удаление tenant'а с каскадом и soft-retention платежей; DPA для B2B.
- Шифрование at rest (диски RDS/MinIO SSE) + TLS in transit (HSTS).

## Тестирование безопасности
- В CI: cross-tenant тесты (пользователь A пытается прочитать/опубликовать ресурсы B → 403/404), fuzzing payload-валидаторов адаптеров, SAST (semgrep), зависимости (npm audit/pip-audit).
- Раз в квартал: review прав облачных/БД, тест восстановления из бэкапа (бэкапы тоже шифруются, содержат PII!).
