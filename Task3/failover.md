# План аварийного переключения и сценарии отказа

## Контекст

Failover проектируется не как единая процедура «переключить всё», а как набор сценариев с разными точками отказа. Критично избегать split-brain и случайного переноса активных поездок в неконсистентное состояние.

## Общая модель failover

1. **Detection.** Health checks, SLO burn-rate alerts, infrastructure signals и synthetic journeys.
2. **Impact classification.** Определяется затронутый слой: AZ, сервис, Kafka, БД, весь регион или внешний провайдер.
3. **Fencing.** Запрещаются новые writes в повреждённом регионе, если есть риск split-brain.
4. **Traffic shift.** DNS/weighted routing и application-level region affinity перенаправляют новые запросы.
5. **Data promotion.** Async replica переводится в primary только после проверки replication lag и fencing token.
6. **State reconstruction.** Redis/hot state/read models перестраиваются из источников истины и компактированных потоков.
7. **Validation.** Synthetic booking/payment/driver journeys проверяют работоспособность.
8. **Controlled return.** После восстановления региона трафик возвращается постепенно, а не автоматически на 100%.

## Принципы

- Новые заказы и активные поездки обрабатываются по разным политикам: новые проще перенаправить, активные требуют восстановления состояния.
- При сомнении в консистентности предпочтителен controlled degradation, а не запись в двух регионах.
- Payment ledger не переписывается; расхождения устраняются через reconciliation и reversing entries.
- Notification/Analytics могут деградировать раньше, чем Booking/Payments.

## Таблица сценариев отказа

| Сценарий | Точка отказа | Обнаружение | Немедленное действие | Переключение | RTO/RPO | Особые риски |
|---|---|---|---|---|---|---|
| Отказ одной AZ | EC2/ALB/DB node в AZ | Multi-AZ health, ошибки 5xx, latency | ALB/ASG выводит узлы AZ; managed DB failover в standby | Внутрирегиональный автоматический failover | RTO 1–5 мин, RPO ≈ 0 для sync replication | Недостаточная capacity в оставшихся AZ |
| Отказ primary PostgreSQL сервиса | DB writer | RDS/Aurora events, connection errors, write failures | Автоматический promotion standby в регионе | Нет межрегионального переключения | RTO 1–5 мин, RPO ≈ 0 | Split-brain при ручном вмешательстве; требуется fencing |
| Деградация Kafka broker/AZ | Broker/ISR | Under-replicated partitions, produce errors | Переназначение leaders, восстановление broker | Внутрирегионально; при полном outage — regional degradation | RTO 5–30 мин, RPO зависит от ISR | Backlog, retry storms, DLQ growth |
| Полный отказ пользовательского региона | Region-wide networking/power/cloud outage | Synthetic checks, health checks, control plane alerts | Fence region, остановить writes, включить incident mode | Перенаправить новые сессии в разрешённый регион; promote DR replicas для critical domains | RTO ≤ 15 мин для Tier0, RPO ≤ 1 мин для async critical data | Активные поездки могут иметь неполное состояние; требуется reconciliation |
| Недоступность глобального control plane | Config/IAM global components | Control plane health, deployment failures | Регионы продолжают работать на последней конфигурации | Не переключать user traffic; восстановить control plane в alternate region | RTO ≤ 60 мин для управления; user impact ≈ 0 | Нельзя создавать новых партнёров/менять глобальные правила |
| Отказ Redis/hot location store | Cache nodes | Eviction/errors, latency, hit ratio | Переход на degraded matching, чтение последних persisted/compacted snapshots | Rebuild hot state из stream | RTO 5–10 мин; потеря stale cache допустима | Кратковременное снижение качества matching |
| Отказ локального PSP | PSP API/bank network | Payment success rate, provider latency | Circuit breaker, предложить другой метод оплаты | Failover на резервного PSP/acquirer, если доступен | RTO 1–10 мин в зависимости от провайдера | Неизвестный статус in-flight авторизаций; status query обязателен |
| Отказ push-провайдера | FCM/APNS/Huawei | Delivery failure rate | Retry/alternate channel; отложить noncritical notifications | Нет regional failover | RTO зависит от провайдера; user impact ограничен | Нельзя терять critical trip notifications; SMS fallback |
| Ошибочный deployment в одном регионе | Новая версия сервиса | Canary metrics, SLO burn rate | Автоматический rollback/canary halt | Нет межрегионального failover, если blast radius ограничен | RTO ≤ 10 мин | Одновременный rollout во всех регионах запрещён |
| DDoS/L7 атака | Edge/WAF/ALB | Traffic spike, WAF events, origin saturation | Включить rate limiting, WAF rules, Shield response, блокировку botnets | Атака локализуется на edge; capacity autoscaling | RTO минуты; RPO неприменим | Ложные блокировки пользователей; требуются allowlists |
| Потеря связности между регионами | Inter-region network | Replication lag, control plane unreachable | Запретить cross-region writes; справочники read-only | Каждая ячейка работает автономно | User impact минимален для hot path | Нельзя выполнять глобальный failover без ручного подтверждения |

## Active trip recovery после регионального отказа

1. Promote booking/payment replicas в DR регионе только после fencing старого региона.
2. Проверить replication lag и последний применённый LSN/event offset.
3. Восстановить saga state и active booking read models.
4. Пересчитать hot driver availability из последних heartbeat/location events.
5. Для поездок с неизвестным состоянием перевести их в `RECOVERY_PENDING` и запретить конфликтующие операции.
6. После подтверждения состояния разрешить продолжение/завершение/отмену.
7. Провести payment reconciliation через PSP status query/settlement reports.

## Failback

- Автоматический failback запрещён.
- Исходный регион сначала работает как replica/catch-up.
- Проверяются data diff, consumer lag, infrastructure capacity и отсутствие повторяющегося сбоя.
- Затем weighted traffic переносится 5% -> 25% -> 50% -> 100%.
- Если ошибки возвращаются, rollback traffic shift выполняется без изменения данных.

## Регулярные проверки

- ежеквартальный региональный failover drill для некритичного трафика;
- ежемесячный restore test PostgreSQL backup;
- еженедельная проверка Kafka consumer lag/DLQ runbooks;
- chaos tests AZ loss и provider timeout;
- tabletop exercise для split-brain и payment unknown-status сценариев.

Диаграмма представлена в [failover-c4.puml](./failover-c4.puml).
