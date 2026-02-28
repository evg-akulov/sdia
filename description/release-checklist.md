# Release Checklist

## Security
-  OIDC scopes проверены
-  Service-to-service JWT rotation выполнена
-  Webhook signature secret актуален
-  Replay-store очищается по TTL

## Observability
-  Alerts проверены
-  Canary metrics заданы в Rollout

## Deployment
-  readiness/liveness настроены
-  preStop hook проверен
-  rollback path протестирован

## Data
-  DB migration — expand выполнен
-  Feature flag включается/выключается
-  Backfill статус OK