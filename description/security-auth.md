# OIDC Authorization Code Flow для UI-клиента

# Token Types

## User JWT

Используется для user-initiated запросов

Содержит:
- sub — user id
- preferred_username
- iss — issuer (Keycloak)
- exp — expiration
- scope / roles

токен храниться у клиента

## Service JWT

Используется для service-to-service вызовов

Получается через:
- OAuth2 Client Credentials Grant

Характеристики:
- sub = service account
- Ограниченные internal scopes

## Таблица доверия

| Caller      | Callee       | Token type  | Context | Где проверяется | Scopes / Roles                                       |
|-------------|--------------|-------------|---------|-----------------|------------------------------------------------------|
| Mobile      | API Gateway  | User JWT    | user    | Gateway         | —                                                    |
| API Gateway | Wallet       | User JWT    | user    | Wallet          | payments.create (Создание / проведение платежа)      |
| API Gateway | CQRS Query Service | User JWT    | user    | CQRS Query Service    | query.read (Просмотр истории и статусов)             |
| Transaction | Wallet       | Service JWT | service | Wallet          | wallet.internal (Internal wallet operations)         |
| Callback    | Transaction  | Service JWT | service | Transaction     | payments.callback (Обработка callback от провайдера) |

- Gateway не выпускает токены
- Gateway только проксирует Authorization header
- Внутренние сервисы:
    - принимают только HTTPS
    - принимают только валидные JWT
    - проверяют scopes строго по endpoint’ам

## Авторизация

Каждый сервис:

Валидирует JWT:
- signature
- expiration
- issuer
- audience

Проверяет:
- наличие нужного scope
- роль (если применимо)


## Failure Modes

Token invalid / expired:
- reject (401)
- Клиент обязан вызывать refresh token

Missing / wrong scope:
- Service возвращает: 403 Forbidden
- Логируется security event

Identity Provider недоступен:
- Уже выданные access tokens продолжают работать
- Refresh невозможен
- Система деградирует, но не падает

Gateway bypass:
- сервисы валидируют JWT
- internal endpoints защищены scopes
- network policy ограничивает ingress

---

# 12-Factor Configuration

Только через environment / secrets

Храним вне кода (secrets и configMaps Kubernetes):
- DB credentials
- OAuth client secrets
- Provider API keys
- Webhook signing secrets
- Feature flags
- Timeout / retry / CB параметры

---

# Поток OAuth2 Client Credentials для M2M (интеграции партнеров/мерчантов).

Шаги:
- Provider отправляет запрос в Api Gateway
- Api Gateway:
- если запрос без токена, то редиректим в IDP
- IDP:
    - проверяет client_id + secret
    - проверяет разрешения
- IDP возвращает access_token
- Provider вызывает Api Gateway с Authorization: Bearer <token>
- Api Gateway:
    - валидирует JWT (signature, exp, aud)
    - пробрасывает запрос в Callback Service
- Callback Service
    - проверяет роли / scope
    -  возвращает ответ