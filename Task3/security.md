# Механизмы защиты и обоснование изменений ландшафта

## Контекст

Глобальная экспансия увеличивает поверхность атаки: публичные API мобильных приложений, партнёрские интеграции, платежи, персональные данные и несколько регионов. Существующей базовой инфраструктурной защиты недостаточно для 99,99% доступности и регуляторных требований.

## Основные изменения в ландшафте

На [C2-диаграмме](./c2-global-security.puml) новые или существенно изменённые элементы выделены красным:

1. **AWS WAF и Shield на региональном периметре** — защита от L7-атак, bot traffic, injection attempts и DDoS.
2. **Network Firewall/egress control** — централизованные правила исходящего трафика и блокировка небезопасных внешних соединений.
3. **Zero trust service mesh** — mTLS и service-to-service authorization вместо доверия к внутренней сети.
4. **Региональные KMS/Secrets Manager** — изоляция ключей и секретов по регионам/классам данных.
5. **Security Hub/GuardDuty/Central SIEM integration** — агрегация security findings и реагирование.
6. **IAM federation и fine-grained authorization** — роли по region/tenant/service, минимальные привилегии.
7. **Private endpoints/internal load balancing** — сервисы и БД не имеют публичного доступа.

## Защита периметра

### WAF

Региональный WAF размещается перед ALB/API Gateway и применяет:

- managed rules для OWASP Top 10;
- rate limiting по IP/device/session;
- правила против credential stuffing;
- geo/rules там, где это не противоречит доступности;
- partner API-specific rate limits и signatures;
- bot mitigation при подтверждённой атаке.

### DDoS protection

AWS Shield Standard покрывает базовые сетевые атаки. Для критичных регионов рекомендуется Shield Advanced, потому что:

- даёт расширенную диагностику и response support;
- интегрируется с WAF и ALB/NLB;
- снижает риск потери availability во время целевой атаки.

### Firewall и egress control

- Network Firewall/security groups/NACL ограничивают входящий и исходящий трафик.
- Сервисы могут обращаться только к approved provider endpoints.
- Доступ БД наружу запрещён; backup/export идёт через controlled endpoints.
- DNS egress фильтруется для prevention data exfiltration.

## Zero trust внутри платформы

Даже внутри VPC нет автоматического доверия:

- каждая рабочая нагрузка получает workload identity;
- service mesh включает mTLS;
- authorization policy определяет, какой сервис может вызывать конкретный API;
- Kafka ACL и Schema Registry permissions выдаются по service account;
- database roles разделены по доменам;
- human access происходит через SSO, MFA и just-in-time elevation.

## Сегментация сети

| Зона | Содержимое | Доступ |
|---|---|---|
| Public edge | WAF, public ALB/NLB | Только HTTPS из интернета |
| Application private subnet | API services, consumers, Flink workers | Не имеет публичных IP; ограниченный egress |
| Data subnet | PostgreSQL, Redis, Kafka brokers, ClickHouse | Только из application subnet по явным security groups |
| Platform subnet | Observability, CI/CD agents, control plane | Строгое RBAC и отдельные policies |
| Partner/integration zone | Partner API/adapters | Изоляция по tenant и rate limits |

## Защита данных

- TLS для всех внешних соединений; mTLS для service-to-service.
- Encryption at rest для PostgreSQL, Kafka, Redis snapshots, ClickHouse, object storage и backups.
- KMS keys разделены минимум по регионам; для regulated tenants — отдельные tenant keys.
- Payment card data не сохраняется в платформе; используется токенизация PSP.
- PII минимизируется в Kafka events и логах.
- Backups шифруются и имеют ограниченный retention в соответствии с compliance profile.

## Управление секретами

- Secrets Manager/Parameter Store или эквивалент в каждом регионе.
- Запрет секретов в env files, образах и CI logs.
- Ротация database credentials и provider keys.
- Access audit и alerting на необычное чтение секретов.
- Для сторонних интеграций — отдельные credentials на tenant/region, чтобы ограничить blast radius.

## Security monitoring и реагирование

- GuardDuty/CloudTrail/security logs поступают в security analytics/SIEM.
- Prometheus/Grafana используются для операционной доступности, но security events идут в отдельный контур с restricted access.
- Алерты на: массовые auth failures, необычные payout изменения, доступ к production data вне runbook, аномальные egress flows, WAF spikes.
- Для инцидентов безопасности существуют отдельные severity levels и legal/compliance escalation.

## Безопасность CI/CD и supply chain

- Образы собираются из approved base images.
- Dependency scanning, container scanning, IaC scanning обязательны в pipeline.
- Подписание образов/артефактов и admission control в Kubernetes.
- GitOps: production изменения только через reviewed merge request.
- Разделение duties: разработчик не может самостоятельно изменить production network/IAM без approval.

## Обоснование выбранного набора

Набор выбран как прагматичный минимум для глобальной платформы:

- WAF/Shield закрывают внешний периметр и поддерживают availability.
- Network segmentation и firewall уменьшают lateral movement и data exfiltration.
- mTLS/zero trust нужен потому, что микросервисная система имеет много внутренних вызовов.
- Региональные KMS/secrets соответствуют compliance и уменьшают blast radius.
- Central security monitoring обязателен из-за распределённого ландшафта.
- Supply chain security критична при быстром выпуске функций каждые две недели.

## Компромиссы

- WAF может создавать false positives; требуется режим обучения и тестирование правил на canary traffic.
- mTLS и service mesh добавляют операционную сложность и небольшую latency, но являются оправданной ценой за изоляцию.
- Строгий egress control затрудняет добавление новых провайдеров; решается через onboarding exception process.
- Централизация security logs должна учитывать data residency; часть событий остаётся региональной.
