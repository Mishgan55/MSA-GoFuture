# Автоматизированный onboarding новых партнёров

## Контекст

Бизнес-цель — запускать партнёрские решения в новых регионах за 2–4 недели. Если каждый tenant создаётся ручными операциями в БД, IAM, Kafka, мониторинге и feature flags, onboarding становится узким местом и источником ошибок. Поэтому процесс автоматизируется как repeatable workflow.

## Целевой процесс

### 1. Заявка и коммерческая конфигурация

В Partner Control Plane создаётся tenant request:

- юридическое название и идентификатор партнёра;
- страны/города запуска;
- желаемый tier изоляции;
- compliance profile;
- продуктовые модули;
- SSO требования;
- ожидаемая нагрузка;
- контактные роли.

### 2. Approval workflow

Заявка проходит автоматические и ручные проверки:

- legal/compliance approval для страны;
- security review для SSO/integration;
- capacity review для высоких нагрузок;
- commercial approval;
- выбор pooled/dedicated tier.

### 3. Tenant provisioning

После approval запускается Temporal workflow:

1. Создать tenant record и уникальный `tenant_id`.
2. Назначить home region и compliance profile.
3. Создать IAM organization/realm/group mappings.
4. Настроить SSO federation или подготовить invitation для tenant admin.
5. Создать database resources:
   - pooled row/RLS registration для standard tier;
   - schema/database/user для dedicated tier.
6. Создать Kafka ACL/topics/consumer groups, если требуется dedicated segregation.
7. Создать object storage prefixes/buckets и encryption key aliases.
8. Создать feature flags и конфигурацию модулей.
9. Настроить PSP/map/push adapters и секреты через approved secrets workflow.
10. Создать rate limits/quotas.
11. Настроить dashboards, alerts и SLO.
12. Создать sandbox environment и seed test data.
13. Включить audit trail и retention policy.

### 4. Валидация

Автоматические проверки:

- tenant admin может войти через SSO;
- tenant API client получает только разрешённые scopes;
- попытка доступа к чужому tenant отклоняется;
- создание заказа/водителя/тарифа в sandbox проходит;
- события содержат корректный `tenant_id`;
- dashboards получают метрики;
- backup/restore/retention policies созданы;
- alerts доставляются правильной команде.

### 5. Pilot и production enablement

- Включается sandbox pilot.
- После приёмки partner operations team получает production access.
- Production rollout выполняется по городам: 1 город -> несколько городов -> полный запуск.
- На период pilot назначается расширенный monitoring и support channel.
- После стабилизации tenant переводится в стандартный operating mode.

## Инструменты автоматизации

| Компонент | Инструмент | Назначение |
|---|---|---|
| Workflow orchestration | Temporal | Долгоживущий onboarding workflow, retries, compensation, visibility |
| Infrastructure provisioning | Terraform/Crossplane | Базы, buckets, IAM bindings, network resources |
| GitOps rollout | ArgoCD | Конфигурация приложений, feature flags, dashboards |
| IAM | Keycloak/OIDC/SAML federation | Tenant admin, SSO, roles, service accounts |
| Secrets | Secrets Manager/KMS | Provider credentials, tenant keys, rotation |
| Validation | Automated smoke/contract tests | Tenant isolation и key user journeys |
| Documentation | Backstage-like developer/partner portal | Статус onboarding и self-service артефакты |

## Управление ошибками provisioning

Temporal workflow хранит состояние шага и выполняет compensation:

- если создание Kafka ACL не удалось, повторяется с backoff;
- если часть ресурсов создана, но approval отменён, выполняется cleanup;
- cleanup не удаляет данные после принятия production traffic без явного approval;
- все failed steps видны оператору с причиной и runbook link.

## Self-service возможности партнёра

После первоначального onboarding партнёр может самостоятельно:

- управлять пользователями и ролями в пределах tenant;
- создавать API clients и webhook endpoints;
- настраивать тарифы/зоны, если разрешено;
- выбирать каналы уведомлений;
- просматривать usage/ SLA dashboards;
- запрашивать повышение tier или добавление региона.

Действия с риском безопасности или стоимости проходят approval.

## SLA onboarding

| Этап | Целевое время после получения всех входных данных |
|---|---:|
| Standard pooled tenant sandbox | ≤ 1 рабочего дня |
| Standard pooled production | 2–5 рабочих дней |
| Dedicated schema/database tenant | 5–10 рабочих дней |
| Dedicated tenant cell/new country | 2–4 недели с учётом compliance и capacity review |

## Компромиссы

- Temporal/control plane становятся критичными для onboarding, но не для hot path поездок.
- Полностью self-service onboarding для regulated tier невозможен из-за legal/security approvals.
- Автоматизация требует строгой идемпотентности и тестирования cleanup, иначе останутся orphan resources.
