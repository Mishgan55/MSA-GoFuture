# Механизмы обеспечения качества данных

## Контекст

ML-модели динамического ценообразования и антифрода чувствительны к пропускам, дублям, задержкам и неправильной интерпретации полей. BI-метрики также должны быть воспроизводимыми. Поэтому качество данных встроено в pipeline, а не выполняется ручной проверкой после инцидента.

## Выбранные механизмы

### 1. Data contracts и Schema Registry

- Каждое событие имеет владельца, версию схемы, compatibility mode и documented semantics.
- Avro-схемы хранятся в Schema Registry.
- CI производителя проверяет backward/forward compatibility.
- Consumer-driven contracts фиксируют ожидания ключевых потребителей.
- Изменение enum/status values считается потенциально breaking даже если технически совместимо.

### 2. Validation на входе

Flink ingestion job проверяет:

- обязательные поля;
- типы и диапазоны значений;
- валидность timestamp и timezone;
- допустимые статусы и enum;
- валюту/сумму;
- наличие `region`, `tenant_id` там, где требуется;
- ссылочную целостность в допустимых пределах.

Невалидные записи не отбрасываются молча: они попадают в quarantine topic/table с причиной ошибки и correlation context.

### 3. Deduplication и identity resolution

- Бизнес-события дедуплицируются по `event_id`.
- Повторная обработка команд проверяет `command_id`/idempotency key.
- Для внешних отчётов используется provider id + ingestion date.
- Для location events допускается замена старой позиции более новой, а не хранение всех дублей в hot aggregates.
- Идентификаторы сущностей проходят canonicalization: одинаковый формат UUID/ULID, явный namespace.

### 4. Data quality rules

Проверки делятся на уровни.

| Уровень | Примеры правил | Реакция |
|---|---|---|
| Critical blocking | отсутствует booking_id, сумма отрицательная, невалидная валюта, нарушена схема | Quarantine, алерт владельцу домена |
| Warning | неожиданно высокий null rate, резкий рост событий, необычный distribution shift | Data quality incident/warning, pipeline продолжает работу |
| Reconciliation | количество completed trips в events не совпадает с OLTP extract/ledger totals | Investigation case, блокировка certified metric при критичном расхождении |
| ML-specific | feature drift, label delay, missing feature rate, training/serving skew | Model quality alert, rollback/fallback model |

### 5. Freshness и completeness SLA

Для каждого dataset определяется:

- owner;
- freshness SLA;
- completeness SLA;
- criticality tier;
- разрешённая задержка источника;
- порог null/duplicates;
- escalation path.

Примеры:

| Dataset | Freshness | Completeness | Комментарий |
|---|---:|---:|---|
| `driver_location_stream` | секунды | ≥ 99,5% ожидаемых активных водителей | Допустимы сетевые пропуски отдельных обновлений |
| `booking_lifecycle_events` | ≤ 2 минут в аналитике | ≥ 99,9% | Critical for funnel |
| `payment_transactions` | ≤ 5 минут | 100% после reconciliation | Финансовые данные не должны иметь пропусков |
| `pricing_zone_surge` | ≤ 1 минуты | ≥ 99,9% активных зон | Используется в pricing feedback |
| `driver_payouts` | T+1 или realtime по требованию | 100% после settlement | Требуется банковская сверка |

### 6. Lineage и каталог

Выбирается OpenLineage-compatible подход и data catalog типа DataHub:

- каждая таблица/topic имеет owner, description, tags, classification;
- lineage связывает source topic -> Flink job -> lakehouse table -> ClickHouse mart -> BI dashboard -> feature/model;
- перед изменением схемы владелец видит downstream consumers;
- certified datasets помечены отдельно от exploratory datasets.

### 7. Тестирование данных

- Unit tests transformation logic.
- Contract tests producer/consumer.
- Data tests в pipeline через Great Expectations/Deequ-like checks.
- Backfill tests на исторической выборке.
- Regression tests после изменения модели/схемы.
- Shadow comparison между legacy reports и новыми витринами на миграционном этапе.

### 8. Observability данных

Метрики качества данных экспортируются в Prometheus/Grafana:

- `data_freshness_seconds{dataset}`;
- `null_ratio{dataset,field}`;
- `duplicate_ratio{dataset}`;
- `schema_validation_errors_total`;
- `quarantine_records_total{reason}`;
- `event_volume_anomaly_score`;
- `reconciliation_diff_total{type}`;
- `feature_drift_score{feature,model}`;
- `training_serving_skew_score{feature}`.

Алерты направляются владельцу dataset, а не только центральной data-команде.

### 9. Reconciliation и audit

Для критичных потоков выполняются регулярные сверки:

- completed trips vs payment capture events;
- payout ledger vs bank/settlement reports;
- notification requests vs provider delivery receipts;
- count of events in Kafka vs lakehouse bronze;
- active trips read model vs Booking source of truth.

Финансовые расхождения не исправляются прямым редактированием данных. Используются correction/reversing entries.

### 10. Управление качеством ML

- Версия модели и версия feature definitions сохраняются при каждом inference.
- Point-in-time join предотвращает data leakage.
- Перед production rollout модель проходит offline evaluation и canary.
- Мониторятся precision/recall fraud model, calibration surge model, conversion impact pricing.
- При деградации качества данных модель переходит на fallback rules или предыдущую stable версию.

## Процесс реагирования на инцидент данных

1. Обнаружение через DQ alert или пользовательский репорт.
2. Определение blast radius: dataset, downstream dashboards, модели, регионы.
3. Пометка affected dataset как degraded/untrusted.
4. Остановка или ограничение downstream publish для certified metrics.
5. Исправление источника/трансформации.
6. Воспроизводимый backfill.
7. Проверка reconciliation и снятие degraded flag.
8. Postmortem для critical data incident.

## Компромиссы

- Полная валидация всех high-volume location events может быть дорогой; используются sampling и агрегированные checks вместо тяжёлой проверки каждого события.
- Слишком строгие blocking rules могут остановить pipeline. Поэтому blocking применяется только к невозможным/опасным данным.
- Каталог и lineage требуют дисциплины команд; без ownership они быстро устаревают.
