# Модель доменных событий

## Общие правила

Событие — это неизменный факт, который уже произошёл в домене-владельце. События используются для интеграции, построения read models, потоковой обработки и аудита. Они не должны содержать лишних персональных данных.

Все события используют **CloudEvents 1.0** envelope:

| Поле | Назначение |
|---|---|
| `id` | Глобально уникальный идентификатор события, UUID/ULID |
| `source` | Сервис-источник, например `//gofuture/booking-service` |
| `type` | Тип, например `com.gofuture.booking.BookingCreated.v1` |
| `subject` | Основная сущность, например `booking/018f...` |
| `time` | Время возникновения факта в UTC |
| `specversion` | Версия CloudEvents |
| `datacontenttype` | `application/avro` или `application/json` для ограниченного числа debugging topics |
| `dataschema` | Ссылка/идентификатор схемы в Schema Registry |
| `region` | Регион возникновения события |
| `tenantid` | Tenant для партнёрских сценариев, если применимо |
| `traceparent` | Контекст распределённой трассировки |

Payload описывается Avro-схемой. JSON допускается только для временных debugging/admin topics, но не как стандарт.

## Классификация событий

1. **Critical lifecycle events** — влияют на состояние активной поездки и платежи.
2. **High-volume telemetry events** — геопозиции и метрики supply/demand.
3. **Business notification events** — запускают уведомления и пользовательские обновления.
4. **Analytics/ML events** — используются downstream для витрин, feature store и отчётности.
5. **Security/fraud events** — сигналы риска и решения антифрода.

## Каталог ключевых событий

| Событие | Производитель | Ключ партиции | Основные поля payload | Основные потребители |
|---|---|---|---|---|
| `BookingRequested.v1` | Booking API/BFF | `booking_id` | booking_id, passenger_id, origin_zone, destination_zone, requested_at, tenant_id | Booking, Fraud, Analytics |
| `BookingCreated.v1` | Booking | `booking_id` | booking_id, passenger_id, region, city_id, pickup/dropoff zone, fare_quote_id, status | Pricing, Fraud, Analytics, Notification |
| `PriceQuoteRequested.v1` | Booking | `booking_id` | quote_id, booking_id, pickup_zone, dropoff_zone, product_class | Pricing |
| `PriceQuoteCalculated.v1` | Pricing | `booking_id` | quote_id, booking_id, base_amount, surge_multiplier, final_amount, currency, expires_at | Booking, Analytics |
| `DriverSearchStarted.v1` | Booking | `booking_id` | booking_id, pickup_zone, radius, product_class | Driver Matching, Analytics |
| `DriverOffered.v1` | Booking | `driver_id` | offer_id, booking_id, driver_id, expires_at, pickup_distance_m | Driver, Analytics |
| `DriverAccepted.v1` | Driver | `booking_id` | booking_id, driver_id, accepted_at, eta_seconds | Booking, Notification |
| `DriverRejected.v1` | Driver | `booking_id` | booking_id, driver_id, reason | Booking, Analytics |
| `DriverLocationUpdated.v1` | Driver Location Ingestion | `driver_id` | driver_id, geo_hash, lat/lon с ограниченной точностью, speed, heading, recorded_at | Matching, Flink supply/demand, Fraud, Analytics |
| `DriverAvailabilityChanged.v1` | Driver | `driver_id` | driver_id, status, shift_id, reason, changed_at | Booking, Flink, Analytics |
| `BookingAssigned.v1` | Booking | `booking_id` | booking_id, driver_id, assigned_at, eta_seconds | Notification, Analytics, Pricing |
| `DriverArrived.v1` | Booking | `booking_id` | booking_id, driver_id, arrived_at, location | Notification, Analytics |
| `TripStarted.v1` | Booking | `booking_id` | booking_id, driver_id, started_at, start_location | Payments, Analytics, Notification |
| `TripCompleted.v1` | Booking | `booking_id` | booking_id, completed_at, distance_m, duration_s, final_amount | Payments, Analytics, Notification |
| `PaymentAuthorizationRequested.v1` | Booking | `payment_id` | payment_id, booking_id, amount, currency, method_ref | Payments |
| `PaymentAuthorized.v1` | Payments | `payment_id` | payment_id, booking_id, amount, authorization_id, authorized_at | Booking, Analytics |
| `PaymentAuthorizationFailed.v1` | Payments | `payment_id` | payment_id, booking_id, failure_code, retryable | Booking, Notification |
| `PaymentCaptured.v1` | Payments | `payment_id` | payment_id, booking_id, captured_amount, ledger_ref | Booking, Analytics |
| `RefundInitiated.v1` | Payments | `payment_id` | refund_id, payment_id, amount, reason | Analytics, Notification |
| `PayoutCreated.v1` | Payments | `driver_id` | payout_id, driver_id, amount, period, bank_ref | Analytics, Notification |
| `FraudRiskScored.v1` | Fraud | `booking_id` | subject_type, subject_id, score, decision, rule_set_version | Booking, Payments, Analytics |
| `SurgeUpdated.v1` | Pricing/Flink | `zone_id` | zone_id, multiplier, valid_from, valid_to, demand_supply_ratio | Pricing, Booking, Analytics |
| `HotZoneDetected.v1` | Flink | `zone_id` | zone_id, driver_density, demand_density, detected_at | Driver, Pricing, Analytics |
| `NotificationRequested.v1` | Notification | `notification_id` | notification_id, recipient_type, template, channel, payload_refs | Notification workers |
| `NotificationDelivered.v1` | Notification | `notification_id` | notification_id, channel, provider, delivered_at | Analytics |
| `NotificationFailed.v1` | Notification | `notification_id` | notification_id, channel, provider, error_code, retryable | Analytics, SRE |

## Правила payload

- `driver_id`, `passenger_id`, `booking_id` — идентификаторы без персональных данных.
- Географические координаты в событиях location округляются до точности, достаточной для matching, если полная точность не требуется по закону/продукту.
- Суммы передаются в minor units и с явной валютой ISO 4217.
- Все временные метки — UTC RFC 3339.
- В payload включается `schema_version` домена, даже при наличии registry.
- События не содержат raw payment card data, адресов в свободной форме и телефонов.

## Жизненный цикл изменения схемы

1. Изменение добавляется как backward-compatible: новые optional fields.
2. Производитель публикует обновлённую схему в Schema Registry.
3. Consumer contract tests подтверждают совместимость.
4. Breaking change выполняется через `.v2` event type и параллельное потребление v1/v2 до конца migration window.
5. После истечения retention/migration window v1 deprecates и закрывается.
