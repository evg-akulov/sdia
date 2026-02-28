# Deployment Strategy

## Общая модель

- GitLab CI отвечает за build и push image, update manifest
- Kubernetes-манифесты хранятся в отдельном Git-репозитории
- Argo Rollouts управляет стратегией деплоя

---

## Rollout Strategy

### Canary deployment

Используем canary-развёртывание с поэтапным увеличением трафика:

| Шаг | Трафик | Пауза                   |
|-----|--------|-------------------------|
| 1   | 10%    | 2 минуты                |
| 2   | 50%    | 5 минут                 |
| 3   | 100%   | после успешного анализа |

---

### Automated analysis

Перед переводом на 100% выполняется анализ метрик Prometheus:
- error rate
- HTTP 5xx

При нарушении SLO:
- rollout автоматически прерывается
- происходит откат на stable revision

---

## Traffic Routing

- Ingress / API Gateway поддерживает weighted routing
- Argo Rollouts управляет весами
- Пользовательский трафик прозрачно распределяется между версиями

---

## Health checks

### Readiness
- `/actuator/health/readiness`
- сервис не получает трафик до полной готовности

### Liveness
- `/actuator/health/liveness`
- автоматический рестарт при зависании

---

## Graceful shutdown

- `preStop: sleep 20s`
- `terminationGracePeriodSeconds: 30`
- позволяет:
    - завершить HTTP-запросы
    - корректно закрыть Kafka consumers
    - избежать потери сообщений

---

## Rollback

- автоматически (по результатам analysis)
- вручную