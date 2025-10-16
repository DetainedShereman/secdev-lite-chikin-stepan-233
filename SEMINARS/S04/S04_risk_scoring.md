# S04 – Risk Scoring (L×I Prioritization)

| Risk ID | Source (DFD/Row)         | Consolidated Description                                  | Threat (S/T/R/I/D/E) | NFR link (ID)    | L (1–5) | Rationale-L                                 | I (1–5) | Rationale-I                             | **Score (=L×I)** | Decision (Top-5?) | ADR candidate                   |
| ------- | ------------------------ | --------------------------------------------------------- | -------------------- | ---------------- | ------- | ------------------------------------------- | ------- | --------------------------------------- | ---------------- | ----------------- | ------------------------------- |
| R-01    | Internet → API           | Повтор или подделка JWT-токена при аутентификации         | S                    | NFR-002, NFR-003 | 4       | Публичный интерфейс, известный вектор       | 5       | Компрометация сессий всех пользователей | **20**           | Top-1             | JWT TTL + Refresh Token         |
| R-02    | Service / Business Logic | SQL-инъекция в динамических запросах                      | T                    | NFR-001, NFR-006 | 3       | Возможна при слабой валидации входа         | 5       | Потеря/изменение данных в БД            | **15**           | Top-2             | Query Parameterization          |
| R-03    | API / Controller         | Утечка PII в логах или ответах                            | I                    | NFR-009, NFR-007 | 3       | Средняя вероятность при ошибках логирования | 4       | Нарушение приватности пользователей     | **12**           | Top-3             | PII Masking / RFC7807           |
| R-04    | Service → External API   | Отсутствие timeout/retry/circuit breaker                  | D                    | NFR-005          | 3       | Средняя частота ошибок внешнего API         | 4       | Зависание сервиса, отказ в обслуживании | **12**           | Top-4             | Circuit Breaker + Retry         |
| R-05    | DB / Database            | Избыточные права DB-пользователя (нет least privilege)    | E                    | NFR-006, NFR-008 | 2       | Редко меняется, ручная настройка            | 5       | Полная компрометация данных             | **10**           | Top-5             | Least Privilege DB Role         |
| R-06    | Internet → API           | DoS на публичном эндпоинте (частые запросы, большие тела) | D                    | NFR-001, NFR-003 | 3       | Публичная поверхность, типовой DoS          | 3       | Кратковременный простой сервиса         | **9**            | –                 | Rate Limiting + Body Size Limit |

---

**Top-5 рисков для последующих ADR (S05):**

1. **R-01 – JWT Replay / Forgery → ADR: “JWT TTL + Refresh Token”**
2. **R-02 – SQL Injection → ADR: “Query Parameterization”**
3. **R-03 – PII Disclosure → ADR: “PII Masking + RFC7807 Errors”**
4. **R-04 – External API Hang (no timeout/retry) → ADR: “Circuit Breaker + Retry”**
5. **R-05 – Excessive DB Privileges → ADR: “Least Privilege DB Role”**
