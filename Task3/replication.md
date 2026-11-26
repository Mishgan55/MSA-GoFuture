# Схема репликации данных между регионами

## Контекст

Глобальная платформа должна одновременно обеспечивать низкую задержку, целостность транзакций и соблюдение резидентности данных. Поэтому не существует одного универсального режима репликации: данные делятся по классам, и для каждого класса выбирается своя модель.

## Принципы

1. **Write-local активных операций.** Активная поездка изменяется только в домашнем регионе.
2. **Synchronous только внутри региона.** Между зонами доступности региона допустима синхронная репликация, между регионами — нет.
3. **Asynchronous cross-region для DR и аналитики.** Между регионами реплицируются только необходимые данные.
4. **Data minimization.** PII не реплицируется глобально без законного основания.
5. **Rebuildable state не реплицируется.** Redis hot cache и ephemeral search indexes восстанавливаются из источников истины.

## Классы данных

| Класс | Примеры | Репликация внутри региона | Межрегиональная репликация | RPO/RTO |
|---|---|---|---|---|
| Активная поездка и Saga state | booking status, active driver assignment | Multi-AZ synchronous/semi-synchronous PostgreSQL | Async replica в назначенный DR region | RPO ≤ 1 мин, RTO ≤ 15 мин |
| Платёжный ledger | transactions, capture/refund/payout | Strong consistency внутри региона, multi-AZ | Continuous backup/WAL archive + async replica; cross-border по compliance rules | Целевой RPO 0 внутри региона; DR RPO ≤ 1 мин |
| Драйверы и профили | driver profile, vehicle, documents refs | Multi-AZ | Только в домашний/разрешённый DR регион | RPO ≤ 5 мин |
| Геопозиции hot state | последняя позиция водителя | Redis replication/in-memory failover | Не реплицируется; восстанавливается из stream/новых обновлений | Допустима потеря устаревших значений |
| Справочники и тарифы | zones, tariffs, feature config | Multi-AZ/versioned storage | Async replication во все затронутые регионы | RPO ≤ 15 мин, допустим controlled rollout |
| Аналитика | события, витрины, ML features | Региональные ClickHouse/object storage | Обезличенные/агрегированные данные в глобальный lakehouse | Freshness minutes/hours по SLA |
| Audit/security | admin actions, payment audit, access logs | Immutable append-only regional storage | Копии в security archive при разрешении закона | RPO ≤ 5 мин |
| Идентификация и партнёрские настройки | tenant config, SSO metadata | Multi-AZ control plane | Репликация control plane между выбранными регионами | RPO ≤ 5 мин |

## Потоки репликации

### PostgreSQL

Для сервисных БД применяется managed PostgreSQL/Aurora PostgreSQL-compatible кластер:

- multi-AZ synchronous/semi-synchronous replication внутри региона;
- автоматический failover primary внутри региона;
- asynchronous cross-region replica только в предопределённый DR-регион;
- PITR через continuous backup/WAL archive;
- отдельные БД/схемы по сервисам без shared database.

DR-реплики не используются для обычного read scaling пользователей из других стран, чтобы не нарушать data residency и не создавать скрытые глобальные зависимости.

### Kafka

- Hot path topics существуют в каждом регионе независимо.
- MirrorMaker 2 реплицирует только:
  - reference topics зон/тарифов;
  - audit topics, разрешённые к централизации;
  - normalized anonymized analytics streams;
  - configuration/catalog events.
- Raw `DriverLocationUpdated` и активные booking/payment topics не реплицируются глобально по умолчанию.
- Для DR активной ячейки допускается целевая репликация critical compacted/snapshot topics в paired DR region.

### Redis/Hot state

- Redis используется как ускоритель и hot state, но не как единственный источник истины.
- Между регионами не реплицируется.
- После failover состояние перестраивается из последних snapshot/compacted events и новых location updates.

### ClickHouse/Object storage

- Региональные витрины хранятся локально.
- Глобальная аналитика получает агрегированные/обезличенные данные.
- Object storage backup копируется cross-region по политике retention и шифрования.

## Маршрутизация failover пар регионов

| Основной регион | Предпочтительный DR/backup контур | Комментарий |
|---|---|---|
| Сингапур | Джакарта или Франкфурт в зависимости от класса данных | Для пользовательских данных SEA предпочтителен SEA DR; глобальные справочники могут использовать Франкфурт |
| Джакарта | Сингапур | Максимально близкий DR в пределах SEA |
| Сан-Паулу | Франкфурт | Альтернативы ограничены; PII переносится только при наличии правового основания и шифрования |
| Франкфурт | Сингапур/Сан-Паулу для control plane replicas | Control plane не является hot path и может восстанавливаться дольше |

## Защита от split-brain

- Запись в регионе активируется только через control plane fencing token/lease.
- DNS failover сам по себе не делает регион writer'ом: writer status проверяется в regional control store.
- При потере связности между регионами каждая ячейка продолжает обслуживать только свои домашние сущности.
- Глобальные справочники переходят в read-only режим при недоступности control plane.
- После восстановления региона выполняется reconciliation перед возвратом write traffic.

## Компромиссы

- Асинхронная репликация может привести к потере последних секунд/минут изменений при катастрофическом отказе региона.
- Строгая резидентность ограничивает выбор DR-региона.
- Для платёжного ledger требуются дополнительные reconciliation и провайдерские settlement reports, потому что техническая репликация не отменяет финансовую сверку.

Схема представлена в [replication.puml](./replication.puml).
