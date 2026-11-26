# Механизмы надёжной доставки событий

## Контекст

В событийной системе недостаточно «отправить сообщение». Нужно гарантировать, что бизнес-эффекты не теряются, не дублируются и могут быть восстановлены после сбоев сервисов, Kafka, баз данных и внешних API.

## Целевая модель

Внутри платформы применяется **effectively-once processing**:

- запись в БД и публикация события не расходятся благодаря transactional outbox;
- Kafka не теряет подтверждённые сообщения при корректно настроенной репликации;
- потребители идемпотентны и безопасно обрабатывают повторы;
- внешние эффекты выполняются через idempotency keys и reconciliation;
- необратимые бизнес-ошибки разрешаются компенсациями, а не удалением истории.

## Producer-side гарантии

### Transactional outbox

1. Сервис в одной локальной транзакции сохраняет бизнес-сущность и запись outbox.
2. Debezium/Kafka Connect читает outbox из WAL.
3. Событие публикуется в Kafka.
4. После подтверждения запись помечается опубликованной или удаляется по retention.

Почему не dual write напрямую: при падении между commit в PostgreSQL и publish в Kafka возникает несогласованность.

### Настройки producer

- `acks=all`;
- `enable.idempotence=true`;
- `retries` достаточные для временных сбоев;
- `max.in.flight.requests.per.connection` совместимое с idempotence/ordering;
- `delivery.timeout.ms` в соответствии с SLA;
- ключ сообщения выбирается по сущности, для которой нужен ordering.

### Ordering

Порядок гарантируется только в пределах партиции. Поэтому события одной поездки используют `booking_id`, а события одного водителя — `driver_id`. Нет попытки обеспечить глобальный ordering по всей системе.

## Broker-side гарантии

- `replication.factor=3`;
- `min.insync.replicas=2`;
- запрет `unclean.leader.election` для critical topics;
- равномерное распределение leaders по broker'ам;
- регулярные broker failure drills;
- мониторинг under-replicated partitions и offline partitions.

## Consumer-side гарантии

### Inbox/deduplication

Каждый критичный consumer сохраняет обработанный `event_id`/`command_id` в inbox-таблице. Повторное событие игнорируется после проверки, что бизнес-эффект уже применён.

Для высокочастотных location events допускается более лёгкая дедупликация по `driver_id + recorded_at` или принятие только более свежей координаты.

### Idempotent business operations

- `booking_id + state_transition` — переход применяется один раз;
- `payment_id + operation_type` — платёжная операция не дублируется;
- `notification_id` — повторная отправка блокируется или заменяется deduped provider request;
- `payout_id` — выплата не создаётся повторно.

### Retry policy

- временные ошибки — retry с exponential backoff и jitter;
- ошибки схемы/validation — non-retryable, в DLQ после ограниченного числа попыток;
- ошибки внешних провайдеров — retry в соответствии с retryable classification;
- retry topics с задержками 5 минут и 30 минут для операций, которым нельзя блокировать основную партицию.

### DLQ

В DLQ сохраняются:

- исходное сообщение;
- причина ошибки;
- stack/consumer metadata;
- время первой и последней попытки;
- `trace_id`, `booking_id` и другой correlation context.

DLQ не является «мусорной корзиной»: для него есть dashboard, ownership, SLA обработки и инструменты replay после исправления.

## Внешние системы

- PSP/bank APIs вызываются с idempotency key.
- Если ответ внешней системы неизвестен, выполняется status query до принятия решения.
- Push-провайдеры дедуплицируют через собственный `notification_id` там, где поддерживается.
- Все внешние вызовы имеют timeout, circuit breaker и rate limiting.

## Backpressure

- Consumers обязаны ограничивать parallelism в соответствии с capacity downstream.
- Flink использует checkpointing и backpressure metrics.
- Location ingestion не выполняет тяжёлую обработку синхронно: принимает, валидирует и публикует в Kafka.
- Если аналитический потребитель перегружен, это не должно останавливать booking hot path.

## Schema validation и poison messages

- Schema Registry compatibility mode: backward для большинства событий.
- Producer не может опубликовать несовместимую схему.
- Consumer валидирует обязательные поля.
- Poison message не блокирует партицию бесконечно — после retry budget он уходит в DLQ.

## Reconciliation

Для критичных потоков существуют периодические reconciliation jobs:

- Booking state vs Payment state;
- Payment ledger vs PSP settlement reports;
- Notification requested vs delivered/failed;
- Saga stuck instances vs timeouts.

Расхождения создают инцидент или support case, а не исправляются незаметно ручными SQL-запросами.

## Что не гарантируется

- Не гарантируется exactly-once внешнего эффекта у провайдера без поддержки idempotency; используется deduplication и reconciliation.
- Не гарантируется глобальный ordering между разными booking/driver.
- Не гарантируется мгновенная консистентность аналитики; для неё действует freshness SLA.

## Тестирование надёжности

Обязательные failure-injection сценарии:

- падение consumer после обработки бизнес-эффекта, но до commit offset;
- restart producer во время публикации outbox;
- потеря одного broker'а и leader election;
- недоступность Schema Registry;
- зависание PSP после списания/авторизации;
- DLQ replay после исправления схемы;
- partition rebalance во время high load.
