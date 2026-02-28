## Принципы миграции

- Zero-downtime — без остановки сервисов и блокирующих операций
- Backward compatible — старые версии сервисов продолжают работать
- Rollback через конфигурацию, без отката схемы БД
- Feature toggle управляет чтением, а не записью
- Schema first — сначала БД, потом код

# Этапы миграции (Expand / Contract)

## Expand — расширение схемы

1. Миграция БД `ALTER TABLE payments ADD COLUMN status_reason VARCHAR(255);`
2. Обновление сервиса сервиса Wallet Service, добавляем feature flag для переключения чтения

## Dual write

Записываем status и status_reason, но status_reason пока не используется

# Switch Read — переключение чтения (Feature Toggle)

Включаем флаг (payments.useStatusReason = true)

- Запись ведется в оба поля (status + status_reason)
- Читаем из поля status_reason
- Флаг влияет только на чтение

# Rollback Plan

Выключаем флаг payments.useStatusReason = false

# Contract

После тщательного тестирования:
- Выполняем бэкап таблицы
- удаляем поле status
- удаляем флаг и всю логику переключения чтения в коде
