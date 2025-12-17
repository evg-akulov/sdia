# Система обработки платежей с eventual consistency, Saga, Transactional Outbox и CQRS

## 1. C4 Container + Component диаграммы

### C4 Container
[container.puml](c4/container.puml)

### C4 Component 
**внутренние компоненты ключевых сервисов**

Wallet Service [wallet_service.puml](c4/wallet_service.puml)

Transaction Service [transaction_service.puml](c4/transaction_service.puml)

Query Service [query_service.puml](c4/query_service.puml)

## 2. Диаграммы последовательности
Основной успешный сценарий
[sequence_success.puml](sequence/sequence_success.puml)

Сценарий с ошибкой
[sequence_error.puml](sequence/sequence_error.puml)

Отдельная диаграмма для flow с callback-service
[sequence_callback.puml](sequence/sequence_callback.puml)

## 3. Описание API контрактов

**Внешние API**

Создание платежа [payment.md](http/payment.md)

callback API провайдера (endpoint Callback Service) [callback.md](http/callback.md)

Запрос статуса платежа (endpoint Query Service) [query.md](http/query.md)


## 4. Описание сущностей (ERD-диаграмма)
**БД Wallet Service**
![wallet_db_erd.png](img/wallet_db_erd.png)        

Описание таблиц:
* wallets - счета с остатками
* users - пользователи системы, владельцы счетов
* wallet_transactions - транзакции по счетам
* wallet_outbox - таблица для outbox pattern

**Redis** хранит paymentId + PaymentResultId для целей дедупликации PaymentResult (с настроенным persistence)

**БД Transaction Service**
![transaction_db.png](img/transaction_db.png)

Описание таблиц:
* payments - транзакции по счетам
* payment_outbox - таблица для outbox pattern

**БД Query Service (read-модель)**

Денормализованное представление ориентированное на чтение в Elasticsearch

![query_db.png](img/query_db.png)

## 5. Описание сообщений для брокера Apache Kafka

Форматы событий: 

PaymentInitiated [payment_initiated.json](events/payment_initiated.json)

PaymentResult PaymentCompleted [payment_result_completed.json](events/payment_result_completed.json)

PaymentResult PaymentFailed [payment_result_failed.json](events/payment_result_failed.json)

ProviderCallbackReceived [provider_callback_received.json](events/provider_callback_received.json)

## 6. Описание процесса исполнения платежа

* От клиента приходит запрос по REST API в API Gateway, проходит проверку JWT токен
* paymentId генерируется на стороне клиента
* API Gateway вызывает Wallet Service по REST API
* Wallet Service в первую очередь дедуплицирует, проверяет по таблице wallet_transactions, не было ли ранее транзакции с таким paymentId, 
такие образом реализуется идемпотентность, если не было такой операции, то проверяет по таблице wallets баланс по счету, если он больше или равен сумме платежа, то продолжаем
* Wallet Service начинает процесс Saga, для записи транзакции и отправки эвента в kafka применяет outbox pattern: в одной 
физической транзакции БД уменьшает баланс в wallets на сумму платежа, создает запись об операции в таблице wallet_transactions в статусе pending и создает запись в wallet_outbox 
записывая event PaymentInitiated в статусе new, в обеих записях первичный ключ payment_id, в этом месте с помощью outbox pattern 
реализуется атомарность записи операции по счету и записи эвента, который нужно отправить в кафку (либо оба записали, либо ни одного)
* Wallet Service с помощью polling вычитывает записи из wallet_outbox в статусе new и отправляет в Apache Kafka event PaymentInitiated,
в случае успешной отправки меняет статус записи на sent
* Payment Query Service получает event PaymentInitiated из Apache Kafka, дедуплицирует его по paymentId таблицы payment_view 
и если такого не было добавляет запись в payment_view
* Transaction Service получает event PaymentInitiated из Apache Kafka и в первую очередь дедуплицирует его, проверяя по 
таблице payments по реквизиту paymentId таким образом реализуется идемпотентность, если не было такого платежа, то продолжаем
* Transaction Service запускает стейт машину (которая проводит операции по статусам) и создает запись об операции в 
таблице payments в статусе new
* Transaction Service вызывает по HTTP API внешнего провайдера, отправляет в него платеж, при успешной отправке меняет статус в payments на sent, 
при ошибке и исчерпании попыток retry - в одной физической транзакции БД: 
1. меняет статус операции в payments на error
2. создает запись в payment_outbox записывая event PaymentResult(PaymentFailed) (формируем в нем PaymentResultId) в статусе new

В обеих записях первичный ключ payment_id, в этом месте с помощью outbox pattern реализуется атомарность смены статуса
   операции и запись эвента, который нужно отправить в кафку (либо оба записали, либо ни одного)
* В Transaction Service стейт машина ожидает подтверждение по операции n duration (настройка) если не дожидается - 
в одной физической транзакции БД:
1. меняет статус операции в таблице payments на error_timeout
2. создает запись в payment_outbox записывая event PaymentResult(PaymentFailed) (формируем в нем PaymentResultId) в статусе new

В обеих записях первичный ключ payment_id, в этом месте с помощью outbox pattern реализуется атомарность смены статуса 
операции и запись эвента, который нужно отправить в кафку (либо оба записали, либо ни одного)
* Callback Service принимает HTTP callback (через API Gateway) от внешнего провайдера, валидирует подпись, дедуплицирует и сохраняет в CallbackDb событие ProviderCallbackReceived в Outbox
* Callback Service с помощью polling вычитывает записи из payment_outbox в статусе new и отправляет в Apache Kafka
* В Transaction Service получает event ProviderCallbackReceived и в одной физической транзакции БД:
1. меняет статус операции в таблице payments на Completed
2. создает запись в payment_outbox записывая PaymentResult(PaymentCompleted) (формируем в нем PaymentResultId) в статусе new

В обеих записях первичный ключ payment_id, в этом месте с помощью outbox pattern реализуется атомарность смены статуса 
операции и запись эвента, который нужно отправить в кафку (либо оба записали, либо ни одного)
* Transaction Service с помощью polling вычитывает записи из payment_outbox в статусе new и отправляет в Apache Kafka 
event PaymentResult(PaymentCompleted/PaymentFailed), в случае успешной отправки меняет статус записи на sent
* Wallet Service получает event PaymentResult(PaymentCompleted/PaymentFailed) из Apache Kafka, дедуплицирует его проверяя 
по ключу paymentId + PaymentResultId в Redis.
Если в Redis такого ключа нету, еще проверим в wallet_transactions по полю payment_result_id.
Если есть в Redis / wallet_transactions.payment_result_id, то это дубль отбрасываем его.
Если нету,то это не дубль, завершаем процесс Saga ранее начатый в Wallet Service.

Кейсы:

_PaymentCompleted_ - одним update меняем статус wallet_transactions на Completed и присваиваем payment_result_id

_PaymentFailed_ - в одной транзакции БД: 
1. wallets.balance увеличиваем на wallet_transactions.amount
2. одним update меняем статус wallet_transactions на Cancelled и присваиваем payment_result_id

Далее после окончания транзакции БД записываем в Redis paymentId + PaymentResultId

Координатором саги является распределенная хореография через события

* Payment Query Service получает event PaymentResult(PaymentCompleted/PaymentFailed) из Apache Kafka, в payment_view обновляет
запись по payment_id
* При запросе на чтение информации о платеже, читаем из Elasticsearch Query Service тем самым реализуем CQRS при котором запросы на
чтение обрабатывает отдельная система со своей БД и reading моделью. Используем Elasticsearch: хорош для полнотекстового поиска, 
фильтрации по полям (искать платежи по имени клиента, статусу, описанию)

