# S04 – STRIDE Matrix


## Элемент 1: Internet → API (публичная поверхность)

| Element / Boundary | Data / Boundary Type | Threat (STRIDE) | Description | NFR link | Mitigation idea |
|--------------------|----------------------|-----------------|-------------|-----------|------------------|
| Internet → API | Auth request / JWT | **S – Spoofing** | Подделка или повтор JWT-токена при входе | **NFR-002**, **NFR-003** | Короткий TTL токена, refresh-токен, проверка полей `iss/aud/exp`, ограничение частоты логинов (rate limit) |
| Internet → API | Request body / JSON | **T – Tampering** | Внедрение произвольных полей или файлов в тело запроса | **NFR-001** | Строгая валидация схем (`Pydantic`), проверка `Content-Type`, лимит размера тела |
| Internet → API | Error responses | **I – Information Disclosure** | Утечка стектрейсов или внутренних ошибок наружу | **NFR-002**, **NFR-007** | RFC7807-ответы без деталей реализации, корреляция ошибок через `correlation_id` |
| Internet → API | Public endpoint | **D – Denial of Service** | Перегрузка сервера большими телами или частыми запросами | **NFR-001**, **NFR-003** | Ограничение RPS, лимит тела запроса ≤ 1 MiB, серверные таймауты |

---

## Элемент 2: API / Controller

| Element / Boundary | Data / Boundary Type | Threat (STRIDE) | Description | NFR link | Mitigation idea |
|--------------------|----------------------|-----------------|-------------|-----------|------------------|
| API / Controller | Internal token / service call | **S – Spoofing** | Проброс доверия без проверки сервисного токена | **NFR-002** | mTLS между сервисами, проверка JWT на уровне контроллера |
| API / Controller | Headers / Request data | **T – Tampering** | Header injection или подмена Host при проксировании | **NFR-001**, **NFR-007** | Нормализация заголовков, whitelist разрешённых header’ов |
| API / Controller | Logs / Metrics | **I – Information Disclosure** | Утечка PII в логах или метриках | **NFR-009**, **NFR-007** | Маскирование email/телефонов, data-classification, redaction |
| API / Controller | Resource limits | **D – DoS / Resilience** | Безграничный пул воркеров вызывает перегрузку | **NFR-004**, **NFR-005** | Лимиты concurrency, back-pressure, graceful degradation |
| API / Controller | Debug routes | **E – Elevation of Privilege** | Доступ к debug/админ эндпоинтам без RBAC | **NFR-006** | RBAC по ролям, feature flags, отключение debug-ручек |

---

## Элемент 3: Service / Business Logic

| Element / Boundary | Data / Boundary Type | Threat (STRIDE) | Description | NFR link | Mitigation idea |
|--------------------|----------------------|-----------------|-------------|-----------|------------------|
| Service | DB credentials | **S – Spoofing** | Сервис использует учётку без разграничений | **NFR-008** | Минимальные привилегии, отдельный пользователь БД |
| Service | SQL queries | **T – Tampering** | SQL-инъекция при формировании динамических фильтров | **NFR-001**, **NFR-006** | Параметризация запросов, ограничение диапазонов |
| Service | Business events | **R – Repudiation** | Нет аудита действий пользователя | **NFR-007**, **NFR-006** | Аудит-таблицы, неизменяемые записи действий |
| Service | External API call | **D – Denial of Service** | Нет таймаутов/ретраев при сбое внешнего API | **NFR-005** | Таймаут ≤ 2 s, максимум 3 retry с джиттером, circuit breaker |
| Service | RBAC checks | **E – Elevation of Privilege** | Пользователь обходит бизнес-логику без проверки роли | **NFR-006** | Проверка tenant/role перед выполнением операций |

---

## Элемент 4: DB / Database

| Element / Boundary | Data / Boundary Type | Threat (STRIDE) | Description | NFR link | Mitigation idea |
|--------------------|----------------------|-----------------|-------------|-----------|------------------|
| DB | Database user rights | **S – Spoofing / EoP** | Учётка приложения имеет избыточные права | **NFR-006**, **NFR-008** | Least privilege, запрет DDL/DROP, отдельная роль |
| DB | Schema / Records | **T – Tampering** | Изменение данных без целостности | **NFR-006**, **NFR-009** | Foreign keys, checksum, миграции с валидацией |
| DB | Logs / Backups | **I – Information Disclosure** | PII без шифрования/маскирования, открытые бэкапы | **NFR-009** | Шифрование, маскирование PII, контроль доступа |
| DB | Query performance | **D – DoS** | Медленные запросы и блокировки таблиц | **NFR-004** | Индексы, лимиты на LIKE, пагинация |
| DB | Tenant isolation | **E – Elevation of Privilege** | Нарушение изоляции между tenant’ами | **NFR-006** | Row-Level Security, tenant_id фильтрация |

---

## Элемент 5: Service → External API

| Element / Boundary | Data / Boundary Type | Threat (STRIDE) | Description | NFR link | Mitigation idea |
|--------------------|----------------------|-----------------|-------------|-----------|------------------|
| Service → External | HTTPS call | **S – Spoofing** | Доверие внешнему API без проверки подлинности | **NFR-005**, **NFR-008** | TLS pinning, проверка CA, валидация issuer |
| Service → External | Response data | **T – Tampering** | Искажённые ответы при слабой валидации схем | **NFR-005** | Проверка схем JSON, подпись ответов |
| Service → External | Logs / Query | **I – Information Disclosure** | Секреты или PII в query string | **NFR-009** | Только в теле/заголовках, redaction логов |
| Service → External | Network latency | **D – DoS** | Нет retry/CB, зависания при сбоях | **NFR-005** | Circuit breaker, retry ≤ 3, таймаут ≤ 2 s |
| Service → External | OAuth token | **E – Elevation of Privilege** | Токен с избыточными правами (over-scoped) | **NFR-006**, **NFR-002** | Минимальные scope, ротация и ревокация токенов |
