# Observability Contract Template

> Скопируйте этот шаблон в `specs/{domain}/spec.md` → раздел `## Observability Contract`.
> Заполните для каждой фичи с runtime-путём.

---

## Observability Contract

### Metrics

| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `http.server.request.duration` | metric (histogram) | каждый HTTP-запрос | `http.method`, `http.route`, `http.status_code`, `trace_id` | Prometheus | нет |
| `[business_metric]` | metric (counter) | [событие] | `[атрибуты]` | Prometheus | нет |

### Traces

| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `[HTTP endpoint]` | span (auto) | обработка запроса | `http.method`, `http.route`, `trace_id` | Jaeger / OTel | нет |
| `[business_operation]` | span (manual) | [момент] | `[атрибуты]` | Jaeger / OTel | **[email: redacted]** |

### Logs

| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `security.pii.redacted` | log (audit) | маскирование PII | `field.name`, `trace_id` | Loki | нет (redacted) |
| `[error_log]` | log (structured) | [событие] | `error_code`, `trace_id` | Loki | **[email: redacted]** |

### SLO

| SLO | Целевое значение | Как измеряется |
|-----|------------------|----------------|
| Latency p95 | ≤ [X]ms | `[metric]{quantile=0.95}` |
| Availability | ≥ [X]% | `(requests - 5xx) / requests` |
| Error rate | ≤ [X]% | `5xx / total requests` |
