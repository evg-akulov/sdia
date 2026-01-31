# Надежность, масштабирование и наблюдаемость платежной системы с Saga, Outbox и CQRS

# 1. Resilience-паттерны

| Взаимодействие                                              | Таймаут | Retries                   | CB    | Bulkhead       | Rate Limit | Fallback  | Обоснование                                                                                                                                                 | 
|-------------------------------------------------------------|---------|---------------------------|-------|----------------|------------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| API Gateway → Wallet Service                                | 3s      | Нет                       | Нет   | Thread pool    | Да         | Нет       | Rate limit c лимитами по клиентскому приложению, защита от всплесков и DDoS атак, отдельный thread pool для обработки запросов, запрос синхронный с timeout |
| Wallet Service -> Transaction Service                       | Нет     | Kafka retry               | Нет   | Consumer group | Нет        | DLQ       | Асинхронное взаимодействие Kafka, ошибки обработки в DLQ                                                                                                    |
| Transaction Service -> Payment Provider (HTTP)              | 3s      | 3 (exp. backoff + jitter) | Да    | Thread pool    | Нет        | PENDING   | Внешний сервис, нужен timeout+retry с backoff,CB, fallback в отложенную обработку                                                                           |
| API Gateway → CQRS Query Service (чтение статуса/истории)   | 3s      | Нет                       | Нет   | Thread pool    | Нет        | Нет       | Read-only операции, есть timeout                                                                                                                            |
| Notifications Service -> внешние провайдеры (SMS/email)     | 3s      | 3 (exp. backoff + jitter) | Да    | client pool    | Нет        | DLQ       | Внешний сервис, нужен timeout+retry с backoff,CB, fallback в DLQ                                                                                            |

# Payment Service → Payment Provider

Ретраим только технические ошибки:
- timeout;
- connection reset;
- HTTP 5xx;
- HTTP 429

Параметры:
- maxAttempts = 3
- backoff = exponential
- jitter = ±30%

Circuit Breaker:
- Sliding window: 50 последних вызовов
- Failure rate threshold: ≥ 50%
- Slow call threshold: ≥ 2s
- Slow call rate threshold: ≥ 50%
- Open state duration: 30 секунд

Half-Open:
- разрешаем 5 пробных запросов;
- при ≥3 ошибке — переключаемся в OPEN;
- при 100% успехе — CLOSED

Fallback:
Если все retry исчерпаны или Circuit Breaker в состоянии OPEN, тогда:
- платёж переводится в PENDING_PROVIDER
- событие отправляется в Kafka;
- клиент получает 202 Accepted

# Notification Service → Notification Provider

Ретраим только технические ошибки:
- timeout;
- HTTP 5xx

Параметры:
- maxAttempts = 5
- backoff = exponential + jitter

---

# 2. Очереди, DLQ и backpressure

## Основные топики

| Топик                  | Назначение                   | Producer            | Consumer                                 |
|------------------------|------------------------------|---------------------|------------------------------------------|
| payments.wallet        | Инициация платежа            | Wallet Service      | Transaction Service, CQRS Query Service  |
| payments.callback      | Получение финального статуса | Callback Service    | Transaction Service                      |
| payments.transactional | Результат обработки платежа  | Transaction Service | Wallet Service, CQRS Query Service       |
| payments.notify        | Отправка уведомлений         | Wallet Service      | Notification Service                     |

## Dead Letter Queue (DLQ)

| Cценарий                                                    | Тип сообщения | Producer            | Consumer                                 | Основной топик         | DLQ                       | Комментарий по DLQ-обработке                                                                                             |
|-------------------------------------------------------------|---------------|---------------------|------------------------------------------|------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Wallet Service → Transaction Service / CQRS Query Service   | command       | Wallet Service      | Transaction Service, CQRS Query Service  | payments.wallet        | payments.wallet.dlq       | Сообщения попадают в DLQ при ошибке маппинга или неконсистентных данных. Разбор ручной с переотправкой после исправления |
| Callback Service → Transaction Service                      | event         | Callback Service    | Transaction Service                      | payments.callback      | provider.callback.dlq     | Сообщения попадают в DLQ при ошибке маппинга. Разбор ручной с переотправкой                                              |
| Transaction Service → Wallet Service / CQRS Query Service   | event         | Transaction Service | Wallet Service, CQRS Query Service       | payments.transactional | payments.transactional.dlq| Сообщения попадают в DLQ при ошибках валидации. Разбор ручной                                                            |
| Wallet Service → Notification Service                       | event         | Wallet Service      | Notification Service                     | payments.notify        | payments.notify.dlq       | Ошибки доставки                                                                                                          |

DLQ мониторятся по:
- количеству сообщений в очереди;

---

## Backpressure и обработка всплесков нагрузки

Ограничение конкуренции консьюмеров:
* Transaction Service - N consumers threads на инстанс 
* Wallet Service - N consumers threads на инстанс
* Notification Service - N consumers threads на инстанс

Ограниченный размер очереди, при заполнении consumer перестаёт забирать новые сообщения

---

# 3. Кэширование

* В read-модель добавляем Redis в качестве кэша, используем Cache-Aside стратегию, тем самым ускоряем ответы при этом в кеше только нужные данные
* Вся справочная информация (мерчанты, тарифы, курсы) кэшируется со стратегией Write-Through, консистентность данных между кэшем и базой
* Точно НЕ кэшируем остатки в Wallet Service, единый источник истины по остаткам только в write-модели

---

# 4. Observability: метрики, логи, трейсы, SLI/SLO

## Стек

Используем стандартный набор технологий:

Prometheus + Grafana (метрики), Loki (логи), OpenTelemetry + Grafana Tempo(трейсы) для сбора метрик, логов и трейсов.

---

## Метрики

API Gateway:
- количество успешных/неуспешных запросов
- max время ответа
- количество ответов > 5s

Wallet Service:
- количество успешных/неуспешных платежей

Transactional Service:
- количество успешных/неуспешных платежей

Callback Service:
- количество успешных/неуспешных платежей

CQRS Query Service:
- количество успешных/неуспешных платежей

Notification Service:
- retry
- circuit breaker metrics

Kafka
- размер очередей

---

# SLI/SLO

| SLI                                | SLO                  | Где меряем                           |
|------------------------------------|----------------------|--------------------------------------|
| Успешность createPayment (2xx/4xx) | ≥ 99.9% за 30 дней   | API Gateway                          |
| p95 latency createPayment          | ≤ 2 сек              | API Gateway + Wallet Service         |
| Доля ошибок провайдера             | ≤ 1% от всех попыток | Transactional Service                |
| Доля платежей в INCONSISTENT       | 0%                   | Wallet Service + Transaction Service |

---

## Дашборды

Operational dashboard по платежам:
- Диаграмма с количеством успешных/неуспешных платежей за N дней
- Таблица с неуспешными платежами и ошибками по ним
- Диаграмма с количеством неуспешных обращений к Provider за последние N дней

---

## 5. Масштабирование и изоляция

# Wallet Service
- Масштабируется горизонтально по RPS
- При росте нагрузки масштабируется число pod-ов

# Transaction Service
- Масштабируется горизонтально как Kafka consumer group
- Количество инстансов ≤ количество партиций топика

# CQRS Query Service
- Масштабируется горизонтально по RPS
- Использует:
    - Redis cache (cache-aside)
    - аналитическую БД (Elasticsearch)

# Notification Service
- Масштабируется горизонтально как Kafka consumer group
- Количество инстансов ≤ количество партиций топика

# Callback Service
- Масштабируется горизонтально для приёма callback-ов от провайдера
- Stateless, устойчив к повторным callback-ам за счёт дедупликации

# Kafka
Масштабирование достигается за счёт:
- Увеличения количества партиций в топиках
- Добавления consumer-ов в consumer group

---

# Разделение потоков и изоляция нагрузки

- Один топик — один тип события
- Consumer Groups:
    - Wallet, Transaction, Query, Notification — независимые группы;
    - сбой одного consumer-а не влияет на других
- Bulkhead-изоляция:
    - отдельные thread pools:
        - для Kafka consumer-ов;
        - для HTTP вызовов провайдера;
        - для работы с БД

---

### Ожидаемый эффект от Bulkhead-изоляции

| Риск                | Как предотвращается                                                                |
|---------------------|------------------------------------------------------------------------------------|
| Каскадный отказ     | Проблема в одном пуле (HTTP к провайдеру) не влияет на другие пулы                 |
| Starvation          | Пулы для DB и Kafka изолированы и всегда имеют гарантированный доступ к ресурсам   |
| Потеря платежей     | Асинхронные потоки продолжают работу даже при деградации внешних систем            |

---

# Учитываемые отказы и способы обработки

Падение одного инстанса сервиса:
- Kubernetes автоматически перезапускает pod
- Трафик перераспределяется балансировщиком

Временная недоступность платежного провайдера:
- Защита:
    - таймауты;
    - retries с backoff;
    - circuit breaker
- Fallback:
    - платеж переводится в PENDING;
    - дальнейшая обработка асинхронно через очередь;

Рост нагрузки на Read-модель:
- Redis снижает нагрузку на Elasticsearch
