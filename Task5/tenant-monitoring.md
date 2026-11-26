# Мониторинг мультитенантного окружения

## Контекст

Мультитенантная платформа должна обнаруживать не только общий отказ сервиса, но и деградацию одного партнёра, noisy neighbour effects, превышение квот и ошибки изоляции. При этом `tenant_id` может создать высокую кардинальность в Prometheus, поэтому модель метрик должна быть контролируемой.

## Принципы

1. **Tenant-aware, но cardinality-safe.** В Prometheus tenant label допускается только для ограниченного числа ключевых tenant/tier метрик или через recording rules; полный контекст хранится в логах/traces.
2. **SLO на tenant tier и отдельные SLA для крупных tenants.**
3. **Usage and cost visibility.** Партнёр и платформа видят потребление ресурсов.
4. **Security/isolation monitoring.** Аномалии доступа и cross-tenant отклонения рассматриваются как security events.
5. **Onboarding observability.** Каждый шаг provisioning имеет статус, latency и error reason.

## Метрики платформы по tenant

### Бизнес/API

- requests rate/error rate/latency по `tenant_tier`, `region` и ограниченному набору `tenant_id`;
- active bookings/drivers по tenant;
- booking success rate и cancellation rate;
- payment authorization success rate;
- notification delivery success;
- API rate limit hits и throttling events;
- webhook delivery failures.

### Изоляция и безопасность

- RLS denial/cross-tenant access attempts;
- JWT tenant mismatch rejections;
- unusual volume export;
- privilege escalation/break-glass activation;
- service account anomalies;
- tenant config changes.

### Ресурсы и noisy neighbour

- CPU/memory/network per namespace/workload с tenant owner mapping;
- database connections/query latency per tenant workload;
- Kafka consumer lag per tenant-specific groups/topics;
- storage usage per tenant prefix/schema/database;
- cost usage per tenant tags/labels.

### Onboarding

- workflow state duration;
- failed provisioning steps;
- time to sandbox/production;
- validation pass/fail rate;
- orphan resources detected.

## Модель cardinality

В Prometheus используются labels:

- обязательные: `service`, `region`, `environment`, `tenant_tier`;
- опциональные controlled: `tenant_id` только для tenants с отдельным SLA или агрегированные recording rules;
- запрещённые: `booking_id`, `driver_id`, `user_id`, `email`, телефон и другие высококардинальные/PII значения.

Полный tenant drill-down выполняется в ClickHouse/log analytics, где high cardinality допустима лучше, чем в Prometheus.

## Dashboards

1. **Tenant overview:** активные заказы, ошибки, latency, usage, payment success, notification success.
2. **Tenant SLA:** availability/latency против contractual SLA, error budget burn.
3. **Tenant isolation/security:** mismatch events, access denials, break-glass, export anomalies.
4. **Capacity/noisy neighbour:** resource consumption, throttling, database hotspots, Kafka lag.
5. **Onboarding pipeline:** tenants by stage, failures, mean time to production.
6. **Cost/showback:** инфраструктурная стоимость tenant/tier/region.

## Алерты

| Severity | Пример |
|---|---|
| Critical | tenant isolation violation или подтверждённый cross-tenant access |
| Critical | критичный tenant недоступен: success rate ниже SLA |
| High | превышение error budget tenant с отдельным SLA |
| High | payment success rate конкретного tenant резко ниже baseline |
| Medium | tenant приближается к rate/resource quota |
| Medium | onboarding workflow застрял на шаге дольше SLA |
| Low | usage/cost anomaly без пользовательского impact |

## Tenant-facing transparency

Партнёр получает собственный portal/dashboard с:

- доступными ему метриками API и бизнес-операций;
- статусом инцидентов;
- usage/quota;
- audit событиями своего tenant;
- статусом onboarding/интеграций.

Партнёр не видит данные других tenants и внутренние платформенные метрики, раскрывающие архитектурные детали или чужую нагрузку.

## Компромиссы

- Слишком детальный tenant monitoring в Prometheus дорог; поэтому используется многоуровневая модель metrics/logs/analytics.
- Tenant-facing dashboards требуют отдельной authorization модели, иначе могут раскрыть чужие данные.
- Noisy neighbour нельзя устранить только мониторингом — требуются quotas, isolation tier и autoscaling.
