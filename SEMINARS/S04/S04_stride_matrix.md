# S04 – STRIDE Matrix

| Element / Boundary      | Data / Boundary Type | Threat (STRIDE) | Description | NFR link | Mitigation idea |
|--------------------------|----------------------|-----------------|-------------|-----------|------------------|
| Internet → API | Auth request / JWT | **S – Spoofing** | Подделка или повтор JWT-токена при входе | **NFR-002**, **NFR-003** | Короткий TTL токена, refresh-токен, проверка полей `iss/aud/exp`, ограничение частоты логинов (rate limit) |
| Internet → API | Request body / JSON | **T – Tampering** | Внедрение произвольных полей или файлов в тело запроса | **NFR-001** | Строгая валидация схем (`Pydantic`), проверка `Content-Type`, лимит размера тела |
| Internet → API | Error responses | **I – Information Disclosure** | Утечка стектрейсов или внутренних ошибок наружу | **NFR-002**, **NFR-007** | RFC7807-ответы без деталей реализации, корреляция ошибок через `correlation_id` |
| Internet → API | Public endpoint | **D – Denial of Service** | Перегрузка сервера большими телами или частыми запросами | **NFR-001**, **NFR-003** | Ограничение RPS, лимит тела запроса ≤ 1 MiB, серверные таймауты |
