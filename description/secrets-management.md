# Secrets: Vault / K8s Secrets и ротация

# Типы секретов

| Секрет              | 	Где используется                                            |
|---------------------|--------------------------------------------------------------|
| Provider API Key	   | Transaction Service                                          |
| Webhook Secret      | API Gateway                                                  |
| OAuth Client Secret | API Gateway / Services                                       |
| DB Credentials	   | Wallet / Transaction / CQRS Query Service / Callback Service |

# Хранение

Vault
- Vault + Kubernetes Auth
- Vault Agent Sidecar
- Secrets → files / env

Kubernetes Secrets
- Base64 encoded
- Mounted as env / volume

# Ротация

Webhook secret
- Gateway принимает:
    - current
    - previous
- Окно совместимости: 24h

Provider API Key
- Новый ключ добавляется в Vault
- Service reloads config
- Старый ключ отзывается