# Схема топиков Kafka и партиционирование

## Контекст

Платформа должна масштабироваться по регионам и не смешивать горячий поток активных поездок с аналитическими replay-потоками. Поэтому применяются отдельные региональные Kafka-кластеры/инсталляции для hot path и правила репликации только необходимых потоков в глобальный аналитический контур.

## Соглашение об именовании

`<env>.<region>.<domain>.<entity>.<event>.v<major>`

Примеры:

- `prod.sea1.booking.booking.created.v1`
- `prod.sea1.driver.location.updated.v1`
- `prod.sa1.pricing.zone.surge-updated.v1`

Для внутридоменных команд Saga:

`<env>.<region>.saga.<flow>.<command>.v<major>`

Например: `prod.sea1.saga.booking.payment-authorization-requested.v1`.

DLQ:

`<original-topic>.dlq`

Retry topics:

`<original-topic>.retry.5m`, `<original-topic>.retry.30m`

## Партиционирование по регионам

Региональный контекст учитывается двумя способами:

1. **Топологически:** у каждого региона есть собственный hot-path Kafka cluster, поэтому активные поездки не зависят от межрегиональной сети.
2. **Логически:** `region` присутствует в имени топика и CloudEvents extension. Это упрощает маршрутизацию, ACL, мониторинг и экспорт в аналитический контур.

Партиционирование внутри региона:

- booking lifecycle — по `booking_id`, чтобы сохранить порядок событий одной поездки;
- driver location — по `driver_id`, чтобы последовательно обрабатывать позиции одного водителя;
- pricing zone events — по `zone_id`;
- payment events — по `payment_id`, а payout batch — по `driver_id`;
- tenant events — по `tenant_id` + entity id в составном ключе.

## Основные топики

| Топик | Партиции на регион | RF | Retention | Cleanup | Назначение |
|---|---:|---:|---|---|---|
| `prod.<region>.driver.location.updated.v1` | 96–192 | 3 | 24 часа | delete | Высокочастотный location stream для matching и Flink |
| `prod.<region>.driver.availability.changed.v1` | 48 | 3 | 7 дней | delete | Изменения online/offline/status |
| `prod.<region>.booking.booking.created.v1` | 48 | 3 | 30 дней | delete | Факт создания заказа |
| `prod.<region>.booking.lifecycle.events.v1` | 64 | 3 | 30 дней | delete | Assigned, Arrived, Started, Completed, Cancelled |
| `prod.<region>.saga.booking.commands.v1` | 48 | 3 | 14 дней | delete | Команды orchestrated Saga |
| `prod.<region>.pricing.quote.calculated.v1` | 48 | 3 | 14 дней | delete | Результаты расчёта цены |
| `prod.<region>.pricing.zone.surge-updated.v1` | 48 | 3 | 14 дней | delete + compact для latest snapshots | Обновления surge по зонам |
| `prod.<region>.payments.transactions.events.v1` | 48 | 3 | 90 дней | delete | Авторизации, capture, refund |
| `prod.<region>.fraud.risk-scored.v1` | 32 | 3 | 30 дней | delete | Risk decisions |
| `prod.<region>.notification.delivery.events.v1` | 32 | 3 | 14 дней | delete | Delivery/failure events |
| `prod.<region>.geography.zone.changed.v1` | 16 | 3 | compact+delete, 30 дней | compact | Справочные изменения зон |
| `prod.<region>.analytics.domain-events.v1` | 64 | 3 | 7 дней | delete | Нормализованный поток в аналитический контур |

Количество партиций — стартовое. Оно выбирается нагрузочным тестированием и не уменьшается после публикации. Увеличение допускается с проверкой ordering impact.

## ACL и ownership

- Каждый топик имеет одного доменного producer-owner.
- Consumers получают read-only ACL на конкретные топики.
- Flink platform получает выделенные service accounts.
- Запрещён wildcard access `*` в production.
- Schema Registry subject ownership совпадает с владельцем домена.

## Репликация и кластеры

### Hot path regional clusters

- Минимум 3 broker'а/зоны доступности в каждом production region.
- `replication.factor=3`, `min.insync.replicas=2`.
- Producer critical topics: `acks=all`, `enable.idempotence=true`, retries включены.
- Не используется synchronous cross-region replication в hot path из-за latency и риска split-brain.

### Cross-region replication

Через MirrorMaker 2 реплицируются:

- справочники зон и тарифов;
- нормализованные события аналитики;
- события аудита, необходимые глобальному control plane;
- обезличенные данные для ML.

Не реплицируются в другие регионы без необходимости:

- raw location events с высокой частотой;
- персональные данные активных поездок;
- payment tokens/данные, попадающие под локальные требования.

## Потребители

| Потребитель | Тип | Топики | Назначение |
|---|---|---|---|
| Booking Saga Orchestrator | Доменный consumer в Booking Service | lifecycle, pricing, payments, fraud | Переводит state machine и запускает компенсации |
| Driver Matching consumer | Доменный consumer | location, availability, booking lifecycle | Поддерживает кандидатов и hot zone avoidance |
| Flink Supply/Demand Job | Stateful stream processing | location, availability, booking created/cancelled | Рассчитывает спрос/предложение по зонам в realtime |
| Flink Surge Calculation Job | Stateful stream processing | supply/demand, quotes | Рассчитывает surge multipliers |
| Flink Hot Zone Detector | CEP/stateful stream processing | location, booking events | Выявляет перегретые зоны и рекомендует перераспределение |
| Notification consumers | Доменные consumers | business events | Создают уведомления, учитывают предпочтения |
| Analytics ingestion | Kafka Connect/Flink | domain events | Пишет в ClickHouse/lakehouse |
| Fraud feature consumers | Consumer/Flink | booking/payment/device signals | Обновляют realtime features и risk scoring |
| Audit consumer | Platform consumer | critical domain events | Неизменяемый audit log |

## Размеры и throughput planning

Базовые допущения для расчёта capacity:

- 500 тысяч активных поездок;
- до 70% поездок имеют online driver stream с обновлением раз в 2–5 секунд;
- пиковый коэффициент 1,5 относительно среднего;
- средний размер location event после Avro/serialization 300–700 байт;
- lifecycle/business events имеют меньший объём, но более строгие требования к сохранности.

Итоговый целевой запас: не менее 250 тысяч location events/с глобально и не менее 50 тысяч business events/с глобально на пике. Кластер sizing уточняется нагрузочными тестами с учётом replication factor и cross-AZ traffic.

## Защита от hotspot

- ключи hash-ируются стандартным Kafka partitioner;
- запрещено использовать `city_id` как единственный ключ для крупных городов — он создаёт перегретые партиции;
- для zone aggregate событий допустим `zone_id`, потому что события уже агрегированы;
- мониторится partition skew: max/median bytes-in и consumer lag;
- при skew ключ детализируется через `driver_id`/`booking_id` или synthetic shard suffix только для append-only telemetry, где порядок сущности не критичен.
