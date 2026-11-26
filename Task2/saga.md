# Saga жизненного цикла заказа

## Контекст

Создание и подтверждение поездки затрагивает несколько доменов: Booking, Pricing, Driver, Payments и Fraud. Распределённая ACID-транзакция между ними невозможна без двухфазного коммита, который неприемлем для высоконагруженной системы. Поэтому применяется Saga.

## Выбор вида Saga

Выбрана **orchestrated Saga**.

Оркестратор находится внутри Booking Service как explicit state machine. Он:

- хранит состояние саги в собственной БД;
- публикует команды;
- обрабатывает события-результаты;
- управляет timeout и retry;
- запускает компенсации;
- предоставляет support-инструменту видимость текущего шага.

### Почему не choreography

Choreography проще для двух-трёх участников, но сценарий заказа имеет ветвления, таймауты, повторный поиск водителя, отмену, платёжные компенсации и fraud decisions. При choreography эта логика расползлась бы по сервисам, а расследование инцидентов было бы сложным. Orchestration даёт единый аудитируемый граф состояний.

### Почему не 2PC/XA

- невозможно использовать между независимыми сервисами и внешними PSP;
- удерживает блокировки и снижает доступность;
- плохо переносит сетевые сбои и горизонтальное масштабирование.

## Основной happy path

1. Пассажир запрашивает поездку.
2. Booking создаёт заказ в статусе `CREATED` и публикует `BookingCreated`.
3. Booking отправляет `PriceQuoteRequested`.
4. Pricing рассчитывает цену и публикует `PriceQuoteCalculated`.
5. Booking запускает fraud pre-check.
6. Fraud публикует `FraudRiskScored` с решением `ALLOW/REVIEW/BLOCK`.
7. Booking запрашивает авторизацию платежа через `PaymentAuthorizationRequested`.
8. Payments публикует `PaymentAuthorized` или `PaymentAuthorizationFailed`.
9. Booking начинает поиск водителя: `DriverSearchStarted`.
10. Driver/Booking matching отправляет предложение: `DriverOffered`.
11. Водитель принимает заказ: `DriverAccepted`.
12. Booking фиксирует `BookingAssigned` и переходит в статус `ASSIGNED`.
13. После прибытия и начала поездки публикуются `DriverArrived`, `TripStarted`.
14. По завершении — `TripCompleted`.
15. Payments выполняет capture и публикует `PaymentCaptured`.
16. Notification отправляет пользовательские уведомления как downstream reaction, не блокируя поездку.

## Диаграмма состояний

```text
REQUESTED
  -> PRICE_PENDING
  -> FRAUD_CHECK_PENDING
  -> PAYMENT_AUTH_PENDING
  -> DRIVER_SEARCHING
  -> DRIVER_OFFERED
  -> ASSIGNED
  -> DRIVER_ARRIVED
  -> IN_PROGRESS
  -> COMPLETED_PENDING_PAYMENT_CAPTURE
  -> COMPLETED
```

## Ключевые ветвления

### Нет доступного водителя

- Оркестратор увеличивает радиус поиска или меняет стратегию matching.
- При использовании smart redistribution система может ожидать более подходящего водителя, а не назначать ближайшего из перегретой зоны.
- Если лимит времени исчерпан: `BookingNoDriverFound`, компенсация `PaymentVoidRequested`, статус `CANCELLED_NO_DRIVER`.

### Водитель не принял или истёк offer timeout

- `DriverOfferExpired`/`DriverRejected`.
- Водитель освобождается из резерва.
- Запускается повторный поиск с учётом rejection reason и hot zone constraints.

### Платёжная авторизация неуспешна

- Booking переходит в `PAYMENT_FAILED`.
- Пользователю предлагается другой способ оплаты.
- Водитель не назначается; если уже был временно зарезервирован, публикуется `DriverReleased`.

### Fraud decision BLOCK

- Заказ отменяется как `CANCELLED_FRAUD`.
- Платёжная авторизация отменяется/не выполняется.
- Создаётся fraud case для расследования.

### Отмена пассажиром или водителем после назначения

- Booking публикует `BookingCancellationRequested`.
- В зависимости от статуса и тарифных правил может быть cancellation fee.
- Payments выполняет void/partial capture/refund.
- Driver освобождается и получает обновлённый статус.

### Ошибка capture после завершения поездки

- Поездка фиксируется как completed по факту, но финансовая сага переводится в `PAYMENT_CAPTURE_PENDING`.
- Запускаются retry по retryable ошибкам PSP.
- При окончательной неудаче создаётся debt collection/manual review case.
- Нельзя «откатывать» факт поездки, потому что услуга уже оказана.

## Компенсации

| Шаг | Компенсирующее действие |
|---|---|
| Резерв водителя | `DriverReleased` |
| Платёжная авторизация | `PaymentVoidRequested` / `PaymentAuthorizationCancelled` |
| Capture выполнен ошибочно | `RefundInitiated` |
| Fraud review блокирует после старта | Manual review + ограничение payout/payment |
| Назначение выполнено, но водитель недоступен | Переназначение или отмена с компенсацией по правилам |
| Уведомление не доставлено | Retry/alternate channel; не является компенсацией поездки |

## Хранение состояния Saga

В Booking DB хранятся:

- `saga_instance_id`;
- `booking_id`;
- текущее состояние;
- список выполненных шагов;
- correlation IDs команд;
- deadline/timeout timestamps;
- retry counters;
- последняя ошибка;
- version для optimistic concurrency.

Изменение состояния и outbox event сохраняются в одной транзакции.

## Таймауты

| Событие ожидания | Timeout | Действие |
|---|---:|---|
| Price quote | 5 секунд | Retry/fallback tariff или отказ с понятной ошибкой |
| Fraud pre-check | 2 секунды для hard decision | При недоступности — правило degraded mode по риску операции |
| Payment authorization | 20 секунд | Timeout query PSP + void/unknown-resolution workflow |
| Driver offer | 15–30 секунд | Offer expired, повторный поиск |
| Общий поиск водителя | 2–5 минут по продукту | Отмена с void |
| Payment capture | До 3 дней retry policy | Manual review/debt workflow |

## Наблюдаемость Saga

Каждый переход состояния генерирует внутреннее событие аудита и telemetry span. В Grafana доступны:

- количество saga по состояниям;
- средняя и p95 длительность полного заказа;
- доля саг, завершившихся компенсацией;
- топ причин отмены;
- stuck sagas без перехода дольше threshold;
- расхождение между booking status и payment status.

## Компромиссы

- Orchestrator становится центральной логикой заказа и требует высокой доступности.
- Необходимо тщательно тестировать все timeout/compensation branches.
- Избыточная сложность для простых вспомогательных процессов; там допускается choreography, но не для payment-critical lifecycle.
