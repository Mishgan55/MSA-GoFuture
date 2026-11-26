# IAM, SSO и матрица ролей мультитенантной платформы

## Контекст

Партнёры должны использовать корпоративный SSO, а платформа — централизованно контролировать доступ по tenant, региону и функции. При этом внутренние команды GoFuture нуждаются в эксплуатационном доступе без нарушения изоляции tenant данных.

## Выбор IAM

Выбирается централизованная IAM-платформа класса **Keycloak** с поддержкой:

- OIDC/OAuth2 для приложений и service accounts;
- SAML/OIDC federation для корпоративных IdP партнёров;
- MFA и conditional access;
- групп/ролей/атрибутов пользователя;
- tenant/region-specific claims;
- audit login/admin events.

### Модель realms/organizations

- `gofuture-internal` — внутренние сотрудники и администраторы платформы.
- `gofuture-partners` или отдельные tenant realms — для партнёров, если требуется сильная изоляция конфигурации SSO.
- Для стандартных партнёров возможна единая партнёрская realm с tenant organizations/claims, если это поддерживается и не снижает изоляцию.
- Для regulated tenants — отдельная realm/конфигурация и tenant-specific keys.

## Модель авторизации

Используется сочетание RBAC и ABAC:

- **RBAC** определяет базовую роль: Tenant Admin, Finance, Support и т.д.
- **ABAC** ограничивает доступ атрибутами: `tenant_id`, `region`, `environment`, `data_classification`, `team`, `purpose`.
- Критичные действия требуют step-up authentication или approval: payout config, изменение тарифов, доступ к PII, экспорт данных.

## Токены и claims

Access token содержит только необходимые claims:

- `sub`;
- `tenant_id`;
- `tenant_tier`;
- `home_region`;
- `roles`;
- `scopes`;
- `auth_time`, `acr` для step-up;
- `session_id` для audit revocation.

PII в токен не помещается. Backend не доверяет `tenant_id` из request body или query params.

## Таблица ролей и уровней доступа

| Роль | Категория | Доступ к tenant данным | Доступ к PII | Операции | Ограничения |
|---|---|---|---|---|---|
| **Platform Owner** | Внутренняя | Нет прямого бизнес-доступа по умолчанию | Нет | Управление платформенными политиками, стандартами, глобальными настройками | Любой доступ к tenant payload требует break-glass |
| **Platform SRE** | Внутренняя | Только технические метаданные/статусы | Нет | Управление инцидентами, capacity, deployments, feature flags | Не видит payload документов/адресов; действия аудируются |
| **Platform Security Admin** | Внутренняя | Security metadata, audit logs | Ограниченно при расследовании | IAM policies, ключи, security findings, блокировки доступа | Доступ к секретам только через controlled процесс |
| **Platform Support L1** | Внутренняя | Ограниченные masked данные конкретного обращения | Только masked/minimized | Просмотр статуса поездки, повторная отправка уведомления, диагностика | Нет payout/tariff изменений; tenant/region scoped |
| **Platform Support L2** | Внутренняя | Расширенный read access по ticket | Ограниченный unmask по approval | Компенсационные операции support workflow, повтор saga step | Time-limited, dual control для чувствительных действий |
| **Tenant Owner/Admin** | Партнёрская | Полный доступ внутри своего tenant | Да, в рамках tenant и роли | Управление пользователями, настройками, интеграциями, тарифами своего tenant | Нет доступа к другим tenants/platform internals |
| **Tenant Operations Manager** | Партнёрская | Операционные данные своего tenant | Ограниченная по необходимости | Управление водителями, зонами, заказами, диспетчеризация | Нет управления SSO/IAM без admin роли |
| **Tenant Finance Manager** | Партнёрская | Платежи, выплаты, отчёты своего tenant | Минимальная | Просмотр/подтверждение payout, reconciliation reports, invoices | Нет изменения технических интеграций и user roles |
| **Tenant Analyst** | Партнёрская | Агрегированные/обезличенные витрины tenant | Нет raw PII | BI dashboards, экспорт разрешённых отчётов | Нет операционных изменений |
| **Tenant Support Agent** | Партнёрская | Данные обращений своего tenant | Ограниченная masked | Помощь пользователям, просмотр статусов, создание tickets | Нет массового экспорта и финансовых операций |
| **Tenant Driver Manager** | Партнёрская | Водители/транспорт своего tenant | Документы только по необходимости | Верификация, активация, блокировка водителей | Нет доступа к пассажирским платежам |
| **Tenant Fraud Analyst** | Партнёрская | Fraud cases tenant | Ограниченная по кейсу | Разбор кейсов, блокировка/разблокировка по workflow | Не изменяет глобальные модели/правила |
| **Tenant API Integration** | Машинная | Только разрешённые API scopes tenant | Нет, кроме необходимых идентификаторов | Server-to-server интеграции, webhooks | Rate limits, IP/secret constraints, отдельные client credentials |
| **Platform Data Engineer** | Внутренняя | Metadata/DQ metrics; raw доступ только по approved pipeline | Нет по умолчанию | Pipeline, schemas, quality rules, lineage | Production data access через controlled break-glass |
| **Platform ML Engineer** | Внутренняя | Feature metadata/aggregates | Нет raw PII по умолчанию | Обучение/оценка моделей | Доступ к pseudonymized features; model deployment через approval |
| **Auditor/Compliance** | Внутренняя/внешняя | Audit logs и reports | Только по scope проверки | Read-only аудит, экспорт доказательств | Time-limited, immutable logging доступа |
| **Break-glass Operator** | Внутренняя аварийная | Временно широкий | Временно по инциденту | Аварийное восстановление/расследование | MFA, dual approval, автоматический expiry, обязательный post-review |

## Принципы выдачи доступа

1. Доступ привязан к конкретному tenant/region, а не «глобально для всех партнёров».
2. По умолчанию выдаётся минимально необходимая роль.
3. Доступ к production PII не должен быть постоянным у широких команд.
4. Любой break-glass имеет expiry и post-incident review.
5. Сервисные аккаунты не используются людьми интерактивно.
6. Увольнение/смена роли пользователя отзывает доступ через federation/group membership.

## SSO onboarding партнёра

1. Партнёр предоставляет metadata своего IdP: OIDC discovery или SAML metadata.
2. GoFuture создаёт federation config в IAM.
3. Настраивается attribute mapping: группы партнёра -> роли GoFuture.
4. Выполняется test login в sandbox.
5. Включается MFA/policy по уровню риска.
6. После production enablement доступ аудируется и пересматривается ежеквартально.

## API authorization

- machine-to-machine использует OAuth2 client credentials;
- отдельный client на партнёра/интеграцию;
- scopes вида `booking.read`, `booking.write`, `driver.read`, `payout.read`;
- webhook signatures и rotation secrets;
- rate limiting и anomaly detection по client_id;
- запрет передачи access token в query string.

## Аудит

В аудит попадают:

- входы и ошибки аутентификации;
- изменение ролей/политик;
- создание/отзыв service accounts;
- доступ к PII/unmask events;
- изменение payout/tariff/tenant config;
- break-glass activation;
- массовые выгрузки.

Audit events хранятся immutable и доступны Auditor/Compliance в read-only режиме.
