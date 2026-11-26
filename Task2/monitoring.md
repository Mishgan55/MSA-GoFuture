# Подход к мониторингу событийной платформы

## Контекст

Event-driven архитектура увеличивает число асинхронных взаимодействий. Без наблюдаемости невозможно понять, где застряла поездка, почему цена не пересчиталась или почему уведомление не отправлено. Мониторинг проектируется как часть архитектуры, а не как постфактум.

## Принципы

1. **Мониторинг от бизнес-процесса к инфраструктуре.** Сначала видна судьба заказа, затем сервис, топик, consumer group, broker и инфраструктура.
2. **Единая correlation model.** В API, событиях, логах и трассах присутствуют `trace_id`, `booking_id`, `driver_id`, `payment_id`, `tenant_id`, `region`.
3. **SLO-based alerting.** Алерт создаётся не на каждую техническую метрику, а на угрозу пользовательскому/бизнес-SLO.
4. **Symptom-based paging.** Pager — за симптомы пользователя: рост failed bookings, stuck sagas, payment failures. Инфраструктурные сигналы используются для диагностики и предупреждения.
5. **Observability as a platform.** Команды получают стандартные dashboards и alerts через шаблоны, а не настраивают всё вручную.

## Инструменты

| Область | Выбранный инструмент | Обоснование |
|---|---|---|
| Метрики | Prometheus | Уже используется, стандарт для exporters и Kubernetes |
| Визуализация | Grafana | Уже используется, объединяет метрики, логи и traces |
| Логи | Loki | Уже используется, хорошо интегрируется с Grafana |
| Трассы | OpenTelemetry + Tempo или Jaeger | Сквозная трассировка sync/async сценариев, vendor-neutral instrumentation |
| Алерты | Alertmanager | Уже используется, routing по командам/severity |
| Kafka monitoring | Kafka Exporter, Strimzi metrics, JMX Exporter | Lag, throughput, broker health, ISR, partition skew |
| Flink monitoring | Flink metrics reporter + Prometheus | Checkpoint duration, backpressure, state size, restart count |
| Schema Registry monitoring | Exporter/HTTP metrics | Compatibility errors, availability, latency |
| Synthetic checks | Blackbox Exporter | Проверка внешних API и публичных endpoint |
| Incident management | Alertmanager + on-call tooling | Escalation, silence, runbook links |

## Метрики верхнего уровня

### Бизнес-SLO

| Метрика | Целевой сигнал |
|---|---|
| `active_trips` | Текущее количество активных поездок по регионам |
| `booking_creation_success_rate` | ≥ 99,9% успешных созданий при валидном запросе |
| `matching_time_seconds` | p50/p95 времени до назначения водителя |
| `driver_acceptance_rate` | Доля принятых предложений |
| `payment_authorization_success_rate` | Доля успешных авторизаций по PSP и регионам |
| `trip_completion_rate` | Доля поездок, дошедших до completed |
| `notification_delivery_success_rate` | Доставляемость по каналам |
| `surge_update_freshness_seconds` | Свежесть расчёта surge по зонам |
| `hot_zone_detection_latency_seconds` | Время от всплеска спроса до события hot zone |

### End-to-end event platform

| Метрика | Назначение |
|---|---|
| `event_end_to_end_latency_seconds{type,consumer}` | Время от business commit до обработки consumer |
| `outbox_publish_lag_seconds{service}` | Отставание outbox CDC |
| `events_in_total{topic}` | Входной поток |
| `events_processed_total{topic,consumer_group}` | Обработанные события |
| `events_failed_total{reason}` | Ошибки валидации/обработки |
| `dlq_messages_total{topic}` | Попадания в DLQ |
| `schema_compatibility_errors_total` | Ошибки публикации/совместимости |
| `duplicate_events_detected_total` | Эффективность идемпотентности |

### Kafka

| Метрика | Назначение |
|---|---|
| `kafka_consumer_group_lag{group,topic,partition}` | Основной сигнал backlog |
| `lag_duration_seconds_estimate` | Перевод lag из offset во время |
| `kafka_server_broker_topic_messages_in_per_sec` | Throughput broker/topics |
| `bytes_in/out_per_sec` | Нагрузка на сеть/disk |
| `under_replicated_partitions` | Риск потери данных |
| `offline_partitions_count` | Критичный сбой доступности |
| `isr_shrinks/expands` | Стабильность replication |
| `partition_skew_ratio` | Неравномерность нагрузки |
| `request_handler_avg_idle_percent` | Насыщение broker |
| `controller_active_count` | Здоровье кластера |

### Consumers и сервисы

| Метрика | Назначение |
|---|---|
| `consumer_processing_duration_seconds` | p95/p99 обработки одного события |
| `consumer_rebalance_total` | Частые ребалансы указывают на нестабильность |
| `consumer_retry_total` | Нагрузка от временных ошибок |
| `inbox_deduplication_hits_total` | Повторные доставки |
| `external_provider_latency_seconds` | Задержка PSP/push/map APIs |
| `circuit_breaker_open_total` | Деградация внешней зависимости |

### Flink

| Метрика | Назначение |
|---|---|
| `flink_jobmanager_job_uptime` | Стабильность job |
| `flink_taskmanager_backpressure` | Узкие места stream processing |
| `checkpoint_duration_ms` | Влияние checkpoint на latency |
| `checkpoint_failed_total` | Риск потери state consistency |
| `state_size_bytes` | Рост state supply/demand/hot zones |
| `num_restarts_total` | Нестабильность job |
| `records_lag`/`source idle time` | Отставание и простои источников |

## Логи и трассы

- Все structured logs в формате JSON с correlation fields.
- События Kafka содержат W3C `traceparent`; consumer продолжает trace при обработке.
- Для batch processing создаются linked spans, чтобы не терять связь с исходными событиями.
- Логи не содержат payment card data, полных адресов, телефонов и access tokens.
- Trace sampling для high-volume location stream снижается, но critical lifecycle tracing сохраняется на высоком уровне.

## Dashboards

1. **Executive/business:** активные поездки, match rate, payment success, availability по регионам.
2. **Booking journey:** funnel от создания до завершения поездки, причины отказов, stuck states.
3. **Event backbone health:** throughput, lag, DLQ, schema errors, outbox lag.
4. **Kafka capacity:** broker/ISR/disk/network/partition skew.
5. **Flink realtime processing:** checkpoint, backpressure, latency, state growth.
6. **Saga operations:** sagas by state, compensation rate, timeout breakdown.
7. **External providers:** PSP, push, maps latency/error rate.
8. **Per-service SLO:** error budget burn rate, availability, latency.

## Алерты

| Severity | Пример | Условие |
|---|---|---|
| Critical | Booking critical consumer остановлен | lag duration > 5 минут или success rate ниже SLO |
| Critical | Payment authorization деградирует | success rate ниже порога 5–10 минут |
| Critical | Offline partitions > 0 | Немедленный page |
| High | DLQ critical topics растёт | > threshold за 5 минут |
| High | Under-replicated partitions > 0 | Устойчиво более 2 минут |
| High | Saga stuck | Более N экземпляров в timeout state |
| Medium | Flink checkpoint failures | Повторяются более 3 раз подряд |
| Medium | Schema compatibility errors | Рост в production deployments |
| Low | Analytics consumer lag | Не влияет на hot path, но угрожает freshness SLA |

## Интеграция инструментов в архитектуру

Инструменты мониторинга отражены на [C2-диаграмме](./c2-event-platform.puml): сервисы и Kafka экспортируют метрики в Prometheus, traces в Tempo/Jaeger, логи в Loki. Grafana объединяет данные, Alertmanager отправляет алерты. Kafka/Flink/Schema Registry имеют отдельные exporters, а synthetic checks проверяют публичные user journeys.

## Компромиссы

- Высокая детализация labels может привести к cardinality explosion; `tenant_id` и `booking_id` запрещены как labels в Prometheus и используются в логах/traces.
- Полное trace sampling location stream слишком дорого; используется adaptive sampling.
- Необходима governance-модель dashboards, иначе Grafana превратится в набор несогласованных панелей.
