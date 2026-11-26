# Модель изоляции данных между партнёрами

## Контекст

Мультитенантность должна сочетать экономическую эффективность pooled-инфраструктуры и строгую изоляцию для регулируемых партнёров. Единый уровень изоляции для всех либо будет слишком дорогим, либо недостаточно безопасным. Поэтому выбирается tiered model.

## Уровни изоляции

### Tier 1. Standard pooled tenant

Используется для большинства партнёров.

**Механизмы:**

- `tenant_id` является обязательным полем всех tenant-owned таблиц и событий;
- PostgreSQL Row-Level Security включена для всех таблиц с tenant данными;
- приложение подключается к БД под ролью без bypass RLS;
- первичные/уникальные ключи tenant-owned сущностей включают `tenant_id` или используют глобально уникальные UUID с обязательным tenant constraint;
- индексы начинаются с `tenant_id` там, где запросы tenant-scoped;
- Redis keys namespaced по tenant;
- Kafka consumer authorization фильтрует tenant events на application level, а для sensitive доменов применяются отдельные topics/ACL;
- object storage paths имеют префикс `tenant/<tenant_id>/` и scoped credentials.

**Преимущества:** низкая стоимость, быстрый onboarding, общий capacity pool.

**Ограничения:** ошибка в RLS/коде может стать критичной, поэтому требуются автоматические tenant isolation tests.

### Tier 2. Dedicated schema/database tenant

Используется для крупных партнёров, повышенной нагрузки или требований аудита.

**Механизмы:**

- отдельная PostgreSQL database или schema;
- отдельные database credentials;
- отдельные backup/restore policies;
- возможны dedicated read replicas;
- отдельные encryption keys или key aliases;
- Kafka topics могут быть выделены по домену/tenant;
- deployment остаётся общим, но configuration routing изолирует tenant data plane.

**Преимущества:** сильная изоляция без полной стоимости отдельной инфраструктуры.

### Tier 3. Dedicated tenant cell

Используется для регулируемых стран, стратегических партнёров или случаев, когда данные нельзя смешивать.

**Механизмы:**

- отдельный Kubernetes namespace или кластерный контур;
- dedicated node groups/compute quotas;
- dedicated Kafka topics/cluster resources;
- dedicated PostgreSQL/Redis/Object storage;
- tenant-specific KMS keys;
- отдельные WAF/rate limiting/API endpoints;
- tenant-specific deployment rollout policy.

**Преимущества:** максимальная изоляция и управляемый blast radius.

**Недостаток:** высокая стоимость и сложность, поэтому применяется по утверждённым критериям.

## Выбор уровня

| Критерий | Pooled | Dedicated DB/schema | Dedicated cell |
|---|---|---|---|
| Небольшой партнёр/пилот | Да | Опционально | Нет |
| Высокая нагрузка/noisy neighbour | Нет/ограниченно | Да | При необходимости |
| Жёсткая резидентность/аудит | Нет | Иногда | Да |
| Требование dedicated encryption keys | Нет | Да | Да |
| SLA с изоляцией инфраструктуры | Нет | Частично | Да |
| Стратегический партнёр с custom rollout | Нет | Иногда | Да |

## Обязательные правила модели данных

1. Каждая tenant-owned сущность имеет `tenant_id`.
2. `tenant_id` не передаётся клиентом как доверенное поле — он извлекается из токена/server-side tenant context.
3. Все репозитории используют tenant-scoped query helpers; прямой SQL без tenant predicate запрещён для tenant-owned таблиц.
4. RLS включена даже если приложение корректно фильтрует tenant.
5. Cross-tenant операции доступны только platform services с отдельной ролью и аудитом.
6. Analytics допускает cross-tenant агрегаты только после обезличивания и при наличии разрешения.
7. Backup/restore тестируется на tenant-level recovery для dedicated tier и на point-in-time recovery для pooled tier.

## Изоляция событий

- Все доменные события содержат `tenantid` extension.
- Для pooled tenants используется общий domain topic, но consumer ACL ограничен по service account, а не tenant user напрямую.
- Для regulated tenants создаются tenant-specific topics или отдельный cluster namespace.
- Schema validation проверяет, что producer не публикует события с чужим `tenant_id`.
- DLQ также tenant-labeled и доступен только авторизованным командам.

## Изоляция кешей и поиска

- Redis keys: `<tenant_id>:<entity>:<id>` с запретом wildcard administrative access.
- Поисковые индексы Elasticsearch разделяются по tenant или используют filtered aliases с дополнительным application-level enforcement.
- Кеши не являются источником истины и не должны обходить RLS.

## Тестирование изоляции

В CI/CD и production smoke tests входят:

- попытка чтения/записи чужой сущности по id;
- проверка RLS под разными database roles;
- проверка отсутствия cross-tenant данных в search/cache;
- Kafka consumer authorization tests;
- object storage path traversal tests;
- negative tests на подмену `tenant_id` в теле запроса;
- аудит доступа platform support в break-glass режиме.

## Переход между уровнями

Onboarding начинается с pooled tier, если нет иных требований. Переход в dedicated выполняется как миграция:

1. snapshot/CDC tenant data;
2. создание dedicated schema/database/cell;
3. validation;
4. кратковременный write freeze для tenant или controlled dual-write;
5. переключение tenant routing;
6. удаление/архивация tenant rows из pooled storage после rollback window.
