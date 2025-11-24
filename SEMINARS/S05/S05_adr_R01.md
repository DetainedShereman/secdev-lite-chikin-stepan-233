# ADR: Centralized Access Policy Validation at Gateway (R-01)

Status: **Proposed**

## Context

Risk: **R-01 “Over-permissive access rules in API-gateway”** (L=4, I=5, Score=20 — №1 в Top-5)
DFD: **Edge Internet → API Gateway → Internal Services** (policy decision at gateway)
NFR: **NFR-AuthZ**, **NFR-API-Security**, **NFR-Audit**, **NFR-Config-Governance**
Assumptions:

* single public API gateway;
* microservices доверяют результатам авторизации gateway;
* политики в YAML/OpenAPI/OPA хранятся централизованно;
* нет client-side MTLS;
* требуется аудит соответствия политикам (регулятор: хранение access logs ≥ 90 дней).

## Decision

Вводим **централизованную, строго типизированную и версионируемую модель access-политик** на уровне API Gateway, включая обязательную валидацию схемы, синхронный PDP (Policy Decision Point), а также запрет на «implicit allow». Все downstream-сервисы принимают только **аттестированный JWT claims set**, выданный gateway после прохождения policy check.

* Policy Model: **deny-by-default**, явные разрешающие правила.
* Policy Storage: Git-backed **versioned policies** с обязательным `schema.json` (OPA/Rego или YAML).
* Policy Validation: pre-deploy **schema validation + static analysis** (mandatory CI job).
* Gateway Behaviour:

  * проверка роли/тенанта/SCIM-атрибутов;
  * добавление `authz_context` в JWT (immutable, signed);
  * логирование решения (`decision=allow|deny`, policy-id, version).
* Scope: **все публичные API** и приватные multi-tenant endpoints.
* Layer: gateway + CI policy pipeline.

## Alternatives

* **Alt A: Политики в каждом сервисе** — отклонено: рассинхрон правил, сложность управления, высокий риск пропущенных проверок.
* **Alt B: Only-role-based checks** — отклонено: недостаточно гранулярно, ломается multi-tenant isolation.
* **Alt C: Runtime policy pull from services** — отклонено: рост латентности и точек отказа.

## Consequences

**Положительные:**

* единая точка управления авторизацией;
* предотвращение “silent allow” и дрейфа политик;
* формальный аудит (policy version → decision log).

**Негативные:**

* усложнение CI/CD (валидация, статический анализ);
* требуется миграция сервисов на доверие к gateway-issued claims.

## DoD / Acceptance

**Given** корректный JWT от IdP и запрос на защищённый endpoint
**When** gateway выполняет policy evaluation
**Then** решение фиксируется в логах, а запрос пропускается или отклоняется строго по политике.

Checks:

* **test:** contract-test `authz_policy_test` проходит для всех endpoints.
* **log:** в access logs присутствует `policy_version`, `decision`, `authz_context`.
* **scan/policy:** CI job `policy-schema-validate` зелёный, статический анализ без ошибок.
* **metric/SLO:** доля отклонённых запросов по политике < **0.5%** от всего трафика (стабильный baseline), отсутствие «unexpected allow».

## Rollback / Fallback

* переключение флага gateway: `authz_mode=legacy` (политики не применяются, только basic role-check);
* мониторинг аномалий по метрике `unexpected_allow` и росту 403;
* откат policy bundle до предыдущей версии в Git (atomic revert).

## Trace

* DFD: Edge Gateway (S04_dfd.md — Public API ingress)
* STRIDE: Tampering/Repudiation (строки для R-01)
* Risk scoring: R-01 — #1 в Top-5 (S04_risk_scoring.md)
* NFR: NFR-AuthZ, NFR-Audit, NFR-Config-Governance
* Issues: SEC-45, SEC-52 (“gateway policy pipeline”, “authz-context claims”)

## Ownership & Dates

Owner: **Security Architecture Lead**
Reviewers: **API Platform Team**
Date created: **2025-11-24**
Last updated: **2025-11-24**

## Open Questions

* включать ли tenant-scoped rate-limits в ту же policy модель?
* нужен ли отдельный PDP сервис при росте нагрузки?

## Appendix

Пример минимальной схемы policy (фрагмент):

```yaml
apiVersion: v1
kind: AccessPolicy
rules:
  - id: "svc.read"
    effect: "allow"
    roles: ["admin", "reader"]
    tenants: ["*"]
```