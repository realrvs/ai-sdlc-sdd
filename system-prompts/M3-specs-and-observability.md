# M3: Specs & Observability Contract

## Формат specs/{domain-name}/spec.md

### Выбор формата
| Тип фичи | Формат |
|----------|--------|
| Пользовательская (user-facing) | Given/When/Then |
| Системная (internal, infra, API) | EARS (SHALL/SHOULD/MUST) |

### Структура
```markdown
## ADDED Requirements
### Requirement: <name> (ID: FR-NNN)
[Описание 1-2 строки]
#### Scenario: <scenario-name>
- **GIVEN** <контекст>
- **WHEN** <действие/условие>
- **THEN** <ожидаемый результат>
```

### Правила нумерации
- Functional: `FR-NNN` (FR-001, FR-002…)
- Non-functional: `NFR-NNN` (NFR-001, NFR-002…)
- Security NFR: `NFR-SEC-NNN` — обязательны, если `threat-model` выявил угрозы
- **Observability NFR: `NFR-OBS-NNN`** — обязательны, если `observability_context` не `not_applicable`

### Dynamic Limits (по домену)
| Domain | FR range | NFR range | NFR-OBS range | NFR-SEC range |
|--------|----------|-----------|---------------|---------------|
| `auth` | 5–15 | 3–8 | 2–5 | 2–5 |
| `api` | 10–25 | 5–10 | 3–6 | 3–8 |
| `data-pipeline` | 15–40 | 5–12 | 3–8 | 2–6 |
| `workflow` | 10–30 | 5–10 | 3–6 | 2–5 |
| `ui` | 5–20 | 3–8 | 1–4 | 1–3 |
| `infra` | 5–15 | 5–10 | 2–5 | 1–4 |
| default | 5–15 | 3–8 | 2–5 | 1–5 |

> Если диапазон превышен — предложить разделение фичи на подфичи с новым domain.

---

## Observability Contract (NEW)

> Заполняется для каждой фичи с runtime-путём.
> Вход для `design.md` (точки инструментирования) и `test-specs` (assertions).

```markdown
## Observability Contract

### Metrics
| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `http.server.request.duration` | metric (histogram) | каждый HTTP-запрос | `http.method`, `http.route`, `http.status_code`, `trace_id` | Prometheus | нет |
| `registration.completed` | metric (counter) | успешная регистрация | `user.tenant_id`, `trace_id` | Prometheus | нет |

### Traces
| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `POST /register` | span (auto) | обработка регистрации | `http.method`, `http.route`, `trace_id` | Jaeger / OTel | нет |
| `otp.generate` | span (manual) | генерация OTP | `otp.ttl_seconds`, `trace_id` | Jaeger / OTel | нет |

### Logs
| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `security.pii.redacted` | log (audit) | маскирование email | `field.name`, `trace_id` | Loki | нет (redacted) |
| `registration.failed` | log (structured) | ошибка регистрации | `error_code`, `trace_id` | Loki | **email: redacted** |

### SLO
| SLO | Целевое значение | Как измеряется |
|-----|------------------|----------------|
| Latency p95 | ≤ 300ms | `http.server.request.duration{quantile=0.95}` |
| Availability | ≥ 99.9% | `(requests - 5xx) / requests` |
| Error rate | ≤ 0.5% | `5xx / total requests` |
```

### Правила Observability Contract
- Каждый FR, создающий runtime-путь, **обязан** иметь ≥1 сигнал
- Для каждого сигнала указаны **обязательные атрибуты** (включая `trace_id`)
- Каждый SLO имеет **измеримый критерий** и привязан к конкретному сигналу
- Если сигнал содержит PII — пометить `pii: true` (вход для threat-model)
- Если фича не создаёт runtime-путь — `observability: not_applicable` с обоснованием
