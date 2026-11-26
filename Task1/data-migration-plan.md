# План миграции данных

## Контекст

В монолите PostgreSQL выполняет роль shared database: таблицы связаны внешними ключами и cross-domain joins. Объёмы данных велики, особенно история поездок, координаты, платёжные операции и события уведомлений. Остановить платформу на часы или дни нельзя, поэтому применяется online migration.

## Общая стратегия

Для каждого домена используется единый migration playbook:

1. **Inventory.** Определить таблицы, владельца, потребителей, объёмы, скорость прироста, PII, retention и зависимости.
2. **Logical split.** Запретить новые cross-domain foreign keys и joins; перевести междоменные ссылки на идентификаторы.
3. **Create target schema.** Создать БД/схему сервиса, права доступа, backup/restore, мониторинг и migration framework.
4. **Initial snapshot.** Выполнить bulk export/import партициями, без блокировки writes.
5. **CDC delta sync.** Подключить Debezium/CDC или transactional outbox для применения изменений после snapshot.
6. **Validation.** Сравнить counts, checksums, sampled records и business invariants.
7. **Dual-write/read switch.** Перевести запись и чтение волнами через feature flags.
8. **Cutover.** Выполнить короткое окно финального переключения — до 5 минут для write path.
9. **Rollback readiness.** Сохранить обратную синхронизацию или freeze plan на время наблюдения.
10. **Decommission.** Удалить legacy доступы, триггеры синхронизации и устаревшие таблицы после истечения rollback window.

## Правила для больших объёмов

- Исторические данные переносятся партициями по времени и региону.
- Активные поездки переносятся отдельно и в последнюю очередь — они требуют минимальной задержки переключения.
- Для location history нет необходимости переносить все raw data в transactional store: история уходит в data lake/ClickHouse, в Driver Service остаётся hot state.
- Миграции выполняются вне часов пик и с лимитами на IOPS/replication lag.
- Все задачи миграции restartable и idempotent.
- Перед cutover проверяется не только количество строк, но и бизнес-правила: отсутствие orphan bookings, завершённых поездок без финального статуса, платежей без ledger entries.

## Матрица миграции по сервисам

| Сервис | Основные данные | Риск | Стратегия миграции | Критерии готовности |
|---|---|---|---|---|
| Notification | device tokens, templates, delivery log, preferences | Средний: потеря токенов влияет на доставку | Snapshot + CDC; delivery history старше 90 дней в архив/ClickHouse; предпочтения переносятся полностью | Доставка push в shadow mode совпадает с монолитом; bounce/error rates не хуже baseline |
| Geography | zones, city config, geofences, provider mappings | Низкий/средний: в основном справочники | Полный export/import, версионирование справочников, короткое окно write freeze для зон | Расчёт ETA/zone lookup совпадает на regression dataset; rollback — переключение adapter обратно |
| Driver | profiles, vehicles, documents, availability, shifts | Высокий: много связей с booking | Snapshot профилей + CDC; availability hot state строится заново из потока; документы переносятся по driver_id batches | Все активные водители имеют корректный статус; нет потерянных документов; latency ingestion соответствует SLA |
| Driver Location | последняя геопозиция, активный hot state | Средний: история огромна, но не нужна в OLTP | Не мигрировать полную историю в OLTP; реплицировать в lakehouse; hot state заполнить из текущего Redis/PostgreSQL snapshot | 99,9% активных водителей имеют fresh location; сервис выдерживает пиковый write throughput |
| Pricing | tariffs, rules, zone multipliers, quotes | Средний: ошибка влияет на цену | Справочники переносятся полностью; quotes — только активные/недавние; правила версионируются | Quote regression tests совпадают; изменение тарифа проходит approval workflow; feature flag rollback |
| Payments | transactions, ledger, payouts, payment methods tokens | Очень высокий: деньги и аудит | Разделить token vault и ledger; snapshot + CDC + continuous reconciliation; финансовый ledger immutable; cutover только после zero-diff reconciliation | Сумма ledger сходится до копейки; нет дублей/пропусков; успешные тесты refund/payout; sign-off Security/Finance/Compliance |
| Fraud | rules, risk scores, cases, device reputation | Средний/высокий: влияние на безопасность | Правила и активные кейсы переносятся полностью; исторические scores — агрегированно/в аналитический контур | Precision/recall на backtest не хуже baseline; realtime decision latency в SLA |
| Booking | bookings, trip states, offers, cancellations, saga state | Очень высокий: ядро бизнеса | Сначала read-only replica/new read model; затем перенос завершённых поездок партициями; активные поездки — в финальном контролируемом окне | Ни одна активная поездка не теряет состояние; state machine проходит replay тесты; rollback не создаёт split brain |
| Analytics | aggregates, reports, event history | Низкий: downstream | Не мигрировать как OLTP; строить заново из Kafka/lakehouse, сверять ключевые агрегаты | Dashboards совпадают в допустимой погрешности; freshness SLA соблюдается |

## Переходные механизмы синхронизации

### Transactional outbox

Сервис-владелец сохраняет бизнес-изменение и outbox-запись в одной транзакции. Debezium публикует outbox в Kafka. Это исключает ситуацию «данные сохранены, но событие потеряно».

### CDC для legacy

Для монолита временно применяется CDC из PostgreSQL. CDC не является долгосрочным public contract: после выделения домена внешние потребители переводятся на canonical domain events.

### Read-through fallback

На время миграции read model нового сервиса может обращаться к legacy facade только для отсутствующих исторических данных. Такие обращения логируются и должны сходить на нет.

## Волны переноса данных

1. **Справочники и конфигурация:** Geography, tariffs, notification templates.
2. **Профили и медленно меняющиеся данные:** Driver, vehicles, documents, payment metadata.
3. **История и downstream данные:** delivery logs, completed trips, analytical aggregates.
4. **Hot state:** availability, active trips, active payment authorizations.
5. **Immutable financial records:** ledger — только после полного reconciliation и контрольного cutover.

## Реестр проверок качества миграции

- полнота: количество записей по ключевым сущностям и partitions;
- точность: hash/checksum выборочных записей, обязательные поля;
- ссылки: отсутствие orphan records;
- бизнес-инварианты: сумма платежей, допустимые статусы поездок, уникальность активного предложения водителю;
- производительность: latency ключевых API до/после;
- операционная готовность: backup/restore drill, monitoring, runbooks, alert coverage.

## Rollback strategy

Rollback зависит от стадии:

| Стадия | Подход |
|---|---|
| Shadow mode | Выключить новый сервис без влияния на пользователей |
| Canary write | Вернуть feature flag на монолит; новые записи синхронизируются обратно или компенсируются |
| После полного cutover | Не выполнять автоматический rollback без оценки расхождений; использовать forward fix и точечные компенсации |
| Финансовые операции | Rollback только через reversing entries/compensation, never destructive update ledger |

## Компромиссы

- CDC и двойная запись временно увеличивают нагрузку и сложность.
- Полный перенос исторических raw location events нецелесообразен, поэтому часть истории доступна только через аналитический контур.
- Короткие write freeze окна возможны для справочников, но не для активных поездок и платежей.
- Расхождения eventual consistency должны объясняться пользователям support-инструментами и статусами обработки.
