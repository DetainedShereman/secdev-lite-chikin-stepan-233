# ADR: Centralized Log Scrubbing & PII-Safe Telemetry (R-04)

Status: **Proposed**

## Context

Risk: **R-04 “Sensitive Data Exposure in Logs & Telemetry”** (L=3, I=5, Score=15 — в Top-5)
DFD: **All Services → Log Sink / APM / Metrics Pipeline**
NFR: **NFR-Logging**, **NFR-Privacy**, **NFR-Observability**, **NFR-Compliance**
Assumptions:

* единый лог-пайплайн (FluentBit → Kafka → ELK/APM);
* микросервисы пишут структурированные логи;
* требования регуляторов: отсутствие персональных данных в техлогах, сохранность 90 дней;
* есть correlation_id на уровне gateway.

## Decision

Вводим **центрированную систему PII-scrubbing и форматирования логов**, включающую обязательные правила маскирования, схему структурированного лога и серверный sidecar/фильтр перед доставкой в Log Sink. Любые сервисы пишут только в **готовый SDK**, который выполняет scrub перед отправкой.

* Policy: запрещено логировать **PII**, токены, пароли, номера карт, email/phone;
* Masking: `email → "***@***"`, `phone → "*******"`, `tokens → "****"`;
* Log schema: обязательные поля `timestamp`, `service`, `env`, `level`, `correlation_id`, `event_id`, `safe_message`;
* Enforcement:

  * CI check `pii-linter` для кода;
  * gateway добавляет `correlation_id`;
  * log-sidecar выполняет финальный scrub (regex + PII-dict).
* Scope: **все сервисы**, все логи уровня `INFO/WARN/ERROR`; исключения — `SEC-AUDIT` (хранятся отдельно).
* Layer: client SDK + sidecar + pipeline filter.

## Alternatives

* **Alt A: Полный запрет логирования динамических данных** — отклонено: снижает observability.
* **Alt B: Scrub только на уровне приложений** — отклонено: возможны пропуски, особенно в сторонних библиотечных логах.
* **Alt C: Соблюдать “manual code review only”** — отклонено: ненадёжно, высокий человеческий фактор.

## Consequences

**Положительные:**

* гарантированное удаление PII из техлогов;
* единая структура логов;
* автоматическое прохождение аудитов.

**Негативные:**

* требуется обновить SDK во всех сервисах;
* небольшой рост latency на sidecar (~1–2 ms).

## DoD / Acceptance

**Given** сервис генерирует логи уровня INFO/ERROR
**When** запись проходит через SDK и sidecar
**Then** в Log Sink отсутствуют поля с PII, все значения проходят маскирование.

Checks:

* **test:** `pii_masking_test` в contract-suite зелёный;
* **log:** логи содержат `correlation_id`, отсутствуют сырые email/phone/tokens;
* **scan/policy:** CI-job `pii-linter` без ошибок;
* **metric/SLO:** доля сообщений, отклонённых или переписанных sidecar < **1%**.

## Rollback / Fallback

* флаг sidecar: `sanitize_enabled=false` (временное отключение при деградации);
* fallback: переход на минимальный вариант SDK-masking только на приложениях;
* мониторинг: `pii_detected` metric (≥ 1 → немедленный rollback).

## Trace

* DFD: Logging pipeline (S04_dfd.md)
* STRIDE: Information Disclosure (строки для R-04)
* Risk scoring: **R-04** — Top-5
* NFR: NFR-Logging, NFR-Privacy, NFR-Observability
* Issues: SEC-66 (“PII scrubber sidecar”), OBS-14 (“Unified log schema”)

## Ownership & Dates

Owner: **Platform Observability Lead**
Reviewers: **Security Engineering**, **Data Privacy Office**
Date created: **2025-11-24**
Last updated: **2025-11-24**

## Open Questions

* включать ли credit-card patterns в high-severity scrub rules?
* нужно ли применять hashing вместо masking для email/phone для аналитики?

## Appendix (optional)

Фрагмент scrub-правил:

```yaml
patterns:
  - name: email
    regex: "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+"
    replacement: "***@***"
  - name: phone
    regex: "\\+?[0-9]{9,15}"
    replacement: "*******"
  - name: token
    regex: "(?:Bearer|Token)\\s+[A-Za-z0-9\\._\\-]+"
    replacement: "****"
```