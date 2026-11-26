# Стратегия соответствия локальным регуляторным требованиям

## Контекст

Выход в Юго-Восточную Азию и Южную Америку означает работу с разными режимами защиты персональных данных, финансового регулирования, локализации и хранения аудита. Стратегия не заменяет юридическую экспертизу по каждой стране, но задаёт архитектурный механизм, позволяющий быстро адаптироваться.

## Принципы

1. **Data residency by design.** Домашний регион определяется страной/городом и tenant configuration.
2. **Data minimization.** В глобальные потоки попадают только необходимые данные; для аналитики — обезличенные или агрегированные.
3. **Regional keys and access.** Ключи шифрования и доступы управляются регионально.
4. **Configurable compliance.** Сроки хранения, consent, доступ и экспорт настраиваются на уровне страны/tenant.
5. **Auditability.** Действия администраторов, доступ к PII и финансовые операции логируются в неизменяемый audit trail.

## Классы данных и правила

| Класс | Примеры | Базовое правило |
|---|---|---|
| Персональные данные пользователей | имя, телефон, email, документы, точные координаты | Хранить в домашнем/разрешённом регионе; не переносить без основания |
| Активные поездки | адреса, маршруты, статусы | Обрабатывать локально; cross-region только для DR по approved policy |
| Платёжные данные | transaction ids, ledger, payment method tokens | Никогда не хранить PAN/CVV; токенизация через PCI DSS-compliant PSP; локальные финансовые требования |
| Данные водителей | документы, банковские реквизиты, лицензии | Строгая резидентность и ограниченный доступ; отдельные retention rules |
| Аналитические данные | агрегаты спроса, средние ETA, зоны | Переносить глобально после псевдонимизации/обезличивания и проверки re-identification risk |
| Аудит и security logs | доступ администраторов, изменения тарифов, auth events | Immutable хранение; централизация только в разрешённом объёме |

## Механизм country/tenant compliance profiles

Для каждой страны и партнёра создаётся machine-readable compliance profile:

- разрешённые регионы хранения;
- запрет/разрешение cross-border transfer;
- сроки retention и удаления;
- требования к consent;
- требования к экспорту данных пользователя;
- требования к локальному аудиту;
- допустимые PSP/map/push-провайдеры;
- необходимость dedicated tenant cell.

Profile применяется через:

- region affinity и routing;
- IAM claims и authorization policies;
- retention jobs;
- encryption/key policy;
- feature flags интеграций;
- data export/anonymization rules.

## Примеры регуляторных режимов

| Юрисдикция | Ключевые соображения | Архитектурное решение |
|---|---|---|
| Индонезия | Требования к локализации и защите персональных данных для отдельных категорий сервисов | Отдельная Jakarta cell; локальные ключи KMS; ограничение cross-region replication PII |
| Бразилия | LGPD: правовые основания, права субъектов данных, прозрачность, удаление/экспорт | São Paulo cell; consent/DSAR tooling; региональный audit; anonymization для analytics |
| Сингапур | PDPA: защита, ограничение retention, разумные security measures | Singapore hub; retention jobs; access audit; минимизация PII в событиях |
| Таиланд/Вьетнам/Малайзия | Различные требования к локализации, consent и хранению | Country profiles; возможность dedicated deployment; запрет глобальной репликации raw PII |
| ЕС/Европейское расширение | GDPR при выходе в Европу | Frankfurt cell, DPIA, DPO processes, SCC/transfer impact assessment для глобальных потоков |

## IAM, доступ и аудит

- Роли поддержки ограничены регионом и tenant; глобальный доступ к PII запрещён по умолчанию.
- Break-glass доступ оформляется временной ролью, двойным подтверждением и обязательным audit case.
- Доступ к платёжным данным разделён: операционный персонал не видит токены/полные реквизиты.
- Все административные изменения тарифов, зон, payout и tenant config записываются в immutable audit.
- Secrets хранятся в regional Secret Manager/KMS; доступ выдаётся workload identity.

## Управление согласиями и правами пользователей

- Consent receipts хранятся в региональном профиле пользователя.
- Запрос на удаление запускает orchestration process по сервисам: profile, booking history, notifications, fraud signals, analytics identifiers.
- Там, где удаление невозможно из-за финансового/налогового audit retention, данные переводятся в restricted retention mode.
- Экспорт данных формируется машиночитаемо и подтверждается владельцем домена.

## Шифрование и ключи

- TLS 1.2+ для внешнего трафика, mTLS внутри service mesh.
- Encryption at rest для БД, Kafka, object storage, backups.
- KMS keys разделены по регионам и классам данных.
- Для premium/regulated tenants возможны dedicated keys и шифрование tenant-specific fields.
- Key rotation и access audit автоматизированы.

## Процесс входа в новую страну

1. Legal/compliance assessment и определение country profile.
2. Выбор режима: использовать существующую ячейку или создать dedicated/local cell.
3. Подключение локальных PSP/maps/notification providers.
4. Настройка retention/consent/audit и IAM policies.
5. Security review, DPIA/transfer assessment при необходимости.
6. Нагрузочное и failover тестирование.
7. Pilot launch с ограниченным городом/tenant.
8. Полный rollout после подтверждения compliance checklist.

## Компромиссы

- Изоляция данных уменьшает точность глобальных ML-моделей; частично компенсируется федеративными агрегатами и privacy-preserving features.
- Dedicated cells дороже, но необходимы для крупных регулируемых рынков.
- Удаление данных в распределённой системе требует orchestration и не может быть мгновенным во всех backups; применяется retention-aware backup policy.
