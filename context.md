# Context

## Feature
User Registration with OTP

## Input Data
```yaml
feature: "User Registration with OTP"
description: "Email-based registration с верификацией через OTP, rate-limit и consent-проверкой"
domain: auth
constraints: ["PostgreSQL", "REST", "OAuth2", "152-ФЗ"]
security_context:
  regulatory: ["152-ФЗ", "GDPR"]
  ai: []
  assets: ["PII", "secrets"]
observability_context:
  stack: ["otel", "loki", "prometheus", "grafana"]
  slo:
    latency_p95_ms: 300
    availability: "99.9%"
    error_rate: "0.5%"
  existing_instrumentation: ["otel-sdk-python"]
```

## Threat Model
- **Методология:** LINDDUN + STRIDE (комбинированно)
- **Обоснование:** regulatory содержит 152-ФЗ/GDPR (LINDDUN), assets содержит PII + auth (STRIDE)
- **Ключевые угрозы:**
  - TM-01 (Risk 15, Critical) — OTP brute-force → NFR-SEC-001
  - TM-02 (Risk 12, High) — PII leak в телеметрию → NFR-SEC-002
  - TM-03 (Risk 10, Medium) — Email enumeration → NFR-SEC-003

## Принятые ADR
- ADR-001: Rate-limiting strategy (hybrid: IP + user_id) — связан с TM-01, NFR-SEC-001
- ADR-002: PII redaction strategy (middleware-based) — связан с TM-02, NFR-SEC-002
- ADR-003: OpenTelemetry auto-instrumentation (hybrid) — связан с NFR-OBS-001

## Согласованные endpoints
- `POST /api/v1/register` — регистрация с OTP
- `POST /api/v1/verify-otp` — верификация OTP

## Observability
- **Стек:** OpenTelemetry, Prometheus, Loki, Grafana
- **SLO:**
  - Latency p95 ≤ 300ms
  - Availability ≥ 99.9%
  - Error rate ≤ 0.5%
- **Ключевые сигналы:**
  - Metrics: `http.server.request.duration`, `registration.completed`, `registration.otp.sent`, `registration.otp.verified`, `registration.otp.failed`, `security.rate_limit.triggered`
  - Logs: `security.rate_limit.triggered`, `security.pii.redacted`, `registration.failed`
  - Traces: `POST /register`, `POST /verify-otp`, `otp.generate`, `otp.verify`, `email.send`
- **Redaction policy:** email маскируется на уровне middleware до экспорта в Loki
- **Корреляция:** W3C `traceparent` из HTTP → OTel spans → Loki/Prometheus; `X-Request-ID` UUID в structlog context

## Открытые вопросы
- [ ] Mail-service поддерживает W3C Trace Context для корреляции?
- [ ] Redis-кластер: dedicated instance или shared с другими сервисами?
- [ ] Нужно ли поддерживать resend OTP через SMS как fallback?
- [ ] TTL OTP: 5 минут — достаточно для UX, или сократить до 3 минут для security?

## Пропущенные артефакты
- (none — all artifacts generated)

## Traceability Matrix (INCREMENTAL)
> Обновляется после каждого артефакта. Полная матрица выводится в конце pipeline.

| ID | Тип | Spec § | TM § | ADR | API | Task IDs | Test IDs | Verify |
|----|-----|--------|------|-----|-----|----------|----------|--------|
| FR-001 | FR | spec.md#req-001 | — | — | /register | T-004, T-005, T-006, T-007 | TC-001, TC-002, TC-003 | — |
| FR-002 | FR | spec.md#req-002 | — | — | /verify-otp | T-011, T-012, T-013 | TC-004, TC-005, TC-006 | — |
| FR-003 | FR | spec.md#req-003 | — | — | /register, /verify-otp | T-014, T-015, T-016 | TC-007, TC-008 | — |
| FR-004 | FR | spec.md#req-004 | — | — | — | T-008, T-009, T-010, T-022, T-023, T-024 | TC-011, TC-012 | — |
| NFR-001 | NFR | spec.md#nfr-001 | — | — | — | T-001 | TC-001 (obs) | SLO p95 ≤ 300ms |
| NFR-002 | NFR | spec.md#nfr-002 | — | — | — | T-001 | — | SLO availability ≥ 99.9% |
| NFR-003 | NFR | spec.md#nfr-003 | — | — | — | — | — | SLO error rate ≤ 0.5% |
| NFR-SEC-001 | NFR-SEC | spec.md#nfr-sec-001 | TM-01 | ADR-001 | — | T-014, T-015 | TC-007, TC-008 | Security test |
| NFR-SEC-002 | NFR-SEC | spec.md#nfr-sec-002 | TM-02 | ADR-002 | — | T-017, T-018 | TC-009 | Security test |
| NFR-SEC-003 | NFR-SEC | spec.md#nfr-sec-003 | TM-03 | — | — | T-006 | TC-010 | Security test |
| NFR-OBS-001 | NFR-OBS | spec.md#nfr-obs-001 | — | ADR-003 | — | T-019, T-020, T-021 | TC-001 (obs), TC-004 (obs) | SLO + signals |
| NFR-OBS-002 | NFR-OBS | spec.md#nfr-obs-002 | — | — | — | T-016, T-024 | TC-007 (obs) | Signal verification |
| TM-01 | TM | — | threat-model.md#tm-01 | ADR-001 | — | T-014, T-015 | TC-007, TC-008 | — |
| TM-02 | TM | — | threat-model.md#tm-02 | ADR-002 | — | T-017, T-018 | TC-009 | — |
| TM-03 | TM | — | threat-model.md#tm-03 | — | — | T-006 | TC-010 | — |
