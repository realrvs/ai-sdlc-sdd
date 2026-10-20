# Threat Model: User Registration with OTP

## 1. Выбор методологии
- **Выбрано:** LINDDUN + STRIDE (комбинированно)
- **Обоснование:** 
  - `security_context.regulatory` содержит 152-ФЗ и GDPR → LINDDUN (приватность)
  - `security_context.assets` содержит PII и secrets → STRIDE (auth, integrity)
  - `security_context.ai` пуст → OWASP LLM не применим
- **Отвергнутые альтернативы:**
  - OWASP Top 10 for LLM — не применим (нет AI/LLM-компонентов)
  - PASTA/OCTAVE — overkill для фичи, не enterprise-масштаб
  - MITRE ATT&CK — нет признаков APT-модели
- **Скоуп:** 
  - `/register` endpoint
  - `/verify-otp` endpoint
  - OTP generation и storage (Redis)
  - Email delivery (mail-service)
  - Telemetry pipeline (OTel → Prometheus/Loki)

## 2. Активы и классификация данных

| Актив | Класс данных | Компонент | Владелец |
|---|---|---|---|
| User email | PII (personal data) | PostgreSQL, Redis | auth-team |
| OTP token (hashed) | secret | Redis | auth-team |
| Consent record | PII (sensitive) | PostgreSQL | compliance-team |
| User request IP | internal | Auth Service | platform-team |
| **Telemetry (logs/traces) (NEW)** | **mixed** | **OTel Collector → Prometheus/Loki** | **platform-team** |
| **OTel spans (email in attributes)** | **PII exposure risk** | **Auth Service → OTel** | **auth-team** |

## 3. Границы доверия (Trust Boundaries)

| ID | Граница | Откуда → Куда | Контроль безопасности |
|----|---------|---------------|----------------------|
| TB-01 | User → Auth Service | untrusted → trusted | Input validation + rate-limit + consent check |
| TB-02 | Auth Service → PostgreSQL | trusted → trusted | Parameterized queries + RBAC |
| TB-03 | Auth Service → Redis | trusted → trusted | Network ACL + token auth |
| TB-04 | Auth Service → Mail Service | trusted → untrusted (external) | TLS 1.3 + API key auth |
| **TB-05 (NEW)** | **Auth Service → Telemetry** | **trusted → trusted** | **PII redaction before export** |
| **TB-06 (NEW)** | **Telemetry → Loki** | **trusted → external storage** | **TLS 1.3 + access control** |

## 4. Визуализация потоков данных

```mermaid
flowchart TD
    subgraph Untrusted ["🔴 Untrusted Zone"]
        User[User Input<br/>(email, otp_code)]
        Attacker[External Attacker]
    end

    subgraph Boundary ["🟡 Trust Boundary Gateways"]
        Sanitizer[Input Sanitizer<br/>+ Validation]
        RateLimiter[Rate Limit Middleware]
        Redactor[PII Redactor<br/>before telemetry]
    end

    subgraph Trusted ["🟢 Trusted Zone"]
        Auth[Auth Service<br/>(FastAPI)]
        DB[(PostgreSQL<br/>(users, consent)]]
        Redis[(Redis<br/>(OTP tokens)]]
    end

    subgraph External ["🟠 External Services"]
        Mail[Mail Service<br/>(OTP email delivery)]
    end

    subgraph Telemetry ["🔵 Telemetry Pipeline"]
        OTel[OTel Collector]
        Prom[Prometheus]
        Loki[Loki]
    end

    User -->|TB-01| Sanitizer
    Sanitizer --> RateLimiter
    RateLimiter --> Auth
    Auth -->|TB-02| DB
    Auth -->|TB-03| Redis
    Auth -->|TB-04| Mail
    Auth -->|TB-05| Redactor
    Redactor --> OTel
    OTel --> Prom
    OTel -->|TB-06| Loki
    Attacker -.->|brute-force| RateLimiter
    Attacker -.->|enumeration| User

    style Untrusted fill:#ffcccc,stroke:#ff0000,stroke-width:2px
    style Boundary fill:#fff2cc,stroke:#ffcc00,stroke-width:2px
    style Trusted fill:#ccffcc,stroke:#00cc00,stroke-width:2px
    style External fill:#ffe0cc,stroke:#ff8800,stroke-width:2px
    style Telemetry fill:#cce5ff,stroke:#0066cc,stroke-width:2px
```

## 5. Матрица угроз

| ID | Категория | Угроза | Вектор | Актив | TB | Risk | Контроль | Requirement |
|----|-----------|--------|--------|-------|----|------|----------|-------------|
| TM-01 | STRIDE-T / LINDDUN-U | OTP brute-force | `/verify-otp` endpoint | OTP tokens | TB-01 | **15 (Critical)** | Rate-limit: 10 attempts/hour per user_id; account lock 30 min | NFR-SEC-001 |
| TM-02 | STRIDE-I / LINDDUN-I | PII leak в телеметрию | log/span attributes | Email в логах и спанах | TB-05 | **12 (High)** | PII redaction middleware; `security.pii.redacted` audit log | NFR-SEC-002 |
| TM-03 | STRIDE-T / LINDDUN-U | Email enumeration | `/register` response timing | User database | TB-01 | **10 (Medium)** | Identical response time for existing/new email; no "user exists" message | NFR-SEC-003 |
| TM-04 | STRIDE-D | OTP token theft | Redis compromise | OTP hashes | TB-03 | **9 (Medium)** | Redis ACL + network isolation; hash storage (not plaintext) | NFR-SEC-001 |
| TM-05 | STRIDE-E | Consent bypass | Request manipulation | Consent record | TB-01 | **8 (Medium)** | Server-side consent validation; immutable consent audit log | NFR-SEC-003 |
| TM-06 | LINDDUN-N | User tracking via telemetry | OTel spans | trace_id correlation | TB-05 | **7 (Low)** | trace_id rotation per request; no persistent user identifiers in metrics | NFR-SEC-002 |

### Risk scoring (DREAD-подобный)
- **Damage:** 1-5 (влияние на пользователя/систему)
- **Reproducibility:** 1-5 (лёгкость воспроизведения)
- **Exploitability:** 1-5 (требуемые навыки атакующего)
- **Risk = Damage × Reproducibility × Exploitability** (max 125, scaled to 1-20)

### Top-3 угрозы
1. **TM-01 (Risk 15)** — OTP brute-force → NFR-SEC-001, P0
2. **TM-02 (Risk 12)** — PII leak в телеметрию → NFR-SEC-002, P0
3. **TM-03 (Risk 10)** — Email enumeration → NFR-SEC-003, P1

## 6. Требования безопасности (→ в spec.md и design.md)

| Requirement | Описание | Покрывает TM | Метрика / verify |
|-------------|----------|--------------|------------------|
| NFR-SEC-001 | Rate-limit OTP: 10 attempts/hour, account lock 30 min | TM-01 | `registration.otp.failed{reason="rate_limit"}`; тест TC-SEC-001 |
| NFR-SEC-002 | PII redaction: email маскируется в логах/спанах до экспорта | TM-02 | 0 PII в логах/спанах; тест TC-SEC-002 |
| NFR-SEC-003 | Email enumeration protection: identical response times | TM-03 | response time delta < 50ms; тест TC-SEC-003 |

## 7. Влияние на ADR
> Какие архитектурные решения должны быть приняты/пересмотрены по итогам TM.

- **ADR-кандидат:** Выбор стратегии rate-limiting (IP-based vs user_id-based) — NFR-SEC-001
- **ADR-кандидат:** Выбор стратегии PII redaction (middleware vs per-service) — NFR-SEC-002
- **ADR-кандидат:** Redis configuration для OTP (dedicated instance vs shared) — TM-04

## 8. Влияние на наблюдаемость

> Как результаты TM влияют на Observability Contract из spec.md.

| Сигнал | Изменение по итогам TM | Связанный TM |
|--------|------------------------|--------------|
| `security.pii.redacted` | добавлен как audit-log при redaction | TM-02 |
| все логи/спаны | PII redaction middleware перед OTel export | TM-02 |
| `http.server.request.duration` | добавить `timing_anomaly` label для enumeration detection | TM-03 |
| экспорт в Loki | TLS 1.3 + access control (TB-06) | TM-02 |
| `registration.otp.failed` | добавить `reason="rate_limit"` для brute-force detection | TM-01 |
