# Карта сервисов и доменные границы

## Контекст

Карта сервисов строится не по техническим слоям, а по bounded contexts. Основой являются существующие продуктовые команды, но введены также платформенные сервисы, без которых доменная архитектура не будет управляемой.

## Доменные сервисы

| Сервис | Команда-владелец | Ответственность | Данные, которыми владеет | Не делает |
|---|---|---|---|---|
| **Booking Service** | Booking | Жизненный цикл заказа, state machine поездки, orchestration Saga, назначение водителя, отмены и компенсации | booking, trip state, cancellation reasons, offer attempts, saga state | Не хранит профили водителей, не считает тарифы, не выполняет платежи |
| **Driver Service** | Driver | Профили водителей и транспорта, документы, статус online/offline, availability, смены | driver profile, vehicle, availability state, work shifts | Не хранит заказы и не выполняет финальный matching |
| **Driver Location Ingestion** | Driver | Приём высокочастотных геопозиций, первичная валидация, публикация в stream | hot location state, последняя известная позиция | Не является системой long-term storage для всех исторических координат |
| **Pricing Service** | Pricing | Базовые тарифы, расчёт стоимости, surge multiplier, динамическое ценообразование, price quote | tariffs, pricing rules, zone multipliers, quotes | Не знает деталей платёжного инструмента и не управляет заказом |
| **Payments Service** | Payments | Платёжные методы, авторизация, capture, refund, payout, ledger, интеграции банков и PSP | payment instruments tokens, transactions, ledger, payouts | Не определяет цену и не принимает решение о назначении водителя |
| **Notification Service** | Notification | Шаблоны, мультиканальная доставка push/SMS/email, device tokens, предпочтения пользователей | notification templates, device tokens, delivery status | Не содержит бизнес-правил заказа; получает уже сформированные события/запросы |
| **Geography Service** | Geography | Геокодирование, маршруты, ETA, зоны спроса, city/region catalog, адаптеры локальных карт | zones, geofences, city configuration, map provider mappings | Не хранит активные поездки и не выполняет billing |
| **Analytics Service** | Analytics | Оперативные витрины, real-time dashboards, агрегаты спроса/предложения, BI API | analytical marts, aggregated metrics, report metadata | Не является источником истины для транзакционных операций |
| **Fraud Service** | Fraud | Risk scoring, правила, device/session signals, блокировки, кейсы для расследования | fraud rules, risk scores, cases, device reputation | Не отменяет платежи самостоятельно; публикует решение/рекомендацию |

## Платформенные сервисы

| Сервис | Владелец | Ответственность |
|---|---|---|
| **API Gateway / Edge** | Platform | Маршрутизация, rate limiting, authentication termination, request validation, version routing, монолит/микросервис coexistence |
| **Identity & Access Management** | Platform | OIDC, SSO, управление пользователями, service accounts, tenant/role/attribute claims |
| **Configuration & Feature Flags** | Platform | Региональная конфигурация, постепенные rollout, аварийные выключатели |
| **Event Backbone** | Platform/Data | Kafka, Schema Registry, connectors, DLQ tooling, событийные стандарты |
| **Service Mesh / Runtime Platform** | Platform | mTLS, service discovery, traffic management, retries, timeouts, observability injection |
| **Observability Platform** | Platform/SRE | Метрики, логи, трассы, алерты, SLO, incident workflows |

## Внешние системы и адаптеры

Интеграции не вызываются напрямую из множества доменов. Для каждого типа внешней системы выделяется адаптер внутри ответственного сервиса:

- платёжные провайдеры и банки — Payments Service;
- Яндекс Карты и локальные картографические сервисы — Geography Service;
- FCM/APNS/Huawei Push Kit и SMS/email-провайдеры — Notification Service;
- партнёрские API — Partner/Open API boundary после внедрения мультитенантности.

## Правила взаимодействия

1. Взаимодействие происходит через public API или доменные события; прямой доступ к чужой БД запрещён.
2. Синхронные цепочки длиннее двух доменных переходов запрещены: Booking не должен синхронно ждать Notification или Analytics.
3. Событие — это факт прошлого, команда — отдельный контракт. Сервисы не должны «подслушивать» чужие внутренние события, не являющиеся public contract.
4. Каждый сервис самостоятельно хранит read model, необходимую для своих операций.
5. Все public events проходят Schema Registry и имеют owner, version, compatibility mode и retention policy.

## Карта зависимостей

- Booking зависит от Driver, Pricing и Payments через команды/события Saga.
- Pricing использует потоки спроса/предложения и geographic zones.
- Fraud получает события Booking/Payments и возвращает risk decisions.
- Notification подписывается на business events и формирует уведомления по шаблонам.
- Geography предоставляет справочные данные зон и маршрутов, но не владеет location stream.
- Analytics является downstream consumer и не участвует в hot path поездки.

## Данные и владение

Каждая сущность имеет единственного владельца:

| Сущность | Владелец |
|---|---|
| Booking/Trip | Booking Service |
| Driver/Vehicle/Availability | Driver Service |
| Last known location | Driver Location Ingestion |
| Tariff/PriceQuote/Surge | Pricing Service |
| PaymentTransaction/Ledger | Payments Service |
| DeviceToken/DeliveryStatus | Notification Service |
| Zone/Route/Geofence | Geography Service |
| RiskScore/FraudCase | Fraud Service |
| Metrics/Aggregates | Analytics Service |

Это владение становится основой для миграции данных и запрета shared database.
