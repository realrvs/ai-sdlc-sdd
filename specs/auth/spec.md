# Spec: User Registration with OTP

## ADDED Requirements

### Requirement: Email registration with OTP (ID: FR-001)
Система создаёт нового пользователя по email с обязательной отправкой OTP-кода на email.

#### Scenario: Successful registration
- **GIVEN** пользователь валидный email и согласие на обработку ПДн
- **WHEN** пользователь отправляет `POST /api/v1/register` с email и consent=true
- **THEN** система создаёт пользователя со статусом `email_unverified`
- **THEN** система генерирует 6-значный OTP, сохраняет хеш в Redis с TTL 5 минут
- **THEN** система отправляет OTP на email через mail-service
- **THEN** система возвращает `201 Created` с `user_id`

#### Scenario: Missing consent
- **GIVEN** пользователь валидный email
- **WHEN** пользователь отправляет `POST /api/v1/register` с consent=false
- **THEN** система возвращает `422 Unprocessable Entity` с `error_code: "consent_required"`

#### Scenario: Duplicate email
- **GIVEN** email уже зарегистрирован в системе
- **WHEN** пользователь отправляет `POST /api/v1/register`
- **THEN** система возвращает `409 Conflict` с `error_code: "email_already_exists"`

---

### Requirement: OTP verification (ID: FR-002)
Пользователь подтверждает email через 6-значный OTP-код.

#### Scenario: Successful OTP verification
- **GIVEN** пользователь с unverified email получил OTP
- **WHEN** пользователь отправляет `POST /api/v1/verify-otp` с валидным кодом
- **THEN** система верифицирует email, устанавливает статус `email_verified`
- **THEN** система удаляет OTP из Redis
- **THEN** система возвращает `200 OK`

#### Scenario: Invalid OTP
- **GIVEN** пользователь с unverified email
- **WHEN** пользователь отправляет `POST /api/v1/verify-otp` с неверным кодом
- **THEN** система возвращает `400 Bad Request` с `error_code: "invalid_otp"`
- **THEN** система инкрементирует счётчик неудачных попыток

#### Scenario: Expired OTP
- **GIVEN** OTP был сгенерирован более 5 минут назад
- **WHEN** пользователь отправляет `POST /api/v1/verify-otp` с этим кодом
- **THEN** система возвращает `410 Gone` с `error_code: "otp_expired"`

---

### Requirement: Rate-limit on registration endpoints (ID: FR-003)
Система ограничивает количество попыток регистрации и верификации.

#### Scenario: Rate limit on registration
- **GIVEN** пользователь отправляет более 3 запросов на `/register` за 1 минуту
- **WHEN** система обрабатывает 4-й запрос
- **THEN** система возвращает `429 Too Many Requests` с `error_code: "rate_limit_exceeded"`
- **THEN** система логирует событие `security.rate_limit.triggered`

#### Scenario: Rate limit on OTP verification
- **GIVEN** пользователь сделал более 10 неудачных попыток верификации за 1 час
- **WHEN** система обрабатывает 11-ю попытку
- **THEN** система блокирует аккаунт на 30 минут
- **THEN** система возвращает `429 Too Many Requests` с `error_code: "account_temporarily_locked"`

---

### Requirement: OTP generation and delivery (ID: FR-004)
Система генерирует криптографически стойкий OTP и доставляет его через email.

#### Scenario: OTP generation
- **GIVEN** успешная регистрация пользователя
- **WHEN** система генерирует OTP
- **THEN** OTP — 6-значный числовой код, сгенерированный через `secrets.token_hex(3)`
- **THEN** система сохраняет хеш OTP в Redis с TTL 300 секунд
- **THEN** система инкрементирует счётчик `registration.otp.sent`

#### Scenario: Email delivery failure
- **GIVEN** OTP сгенерирован и сохранён
- **WHEN** mail-service возвращает ошибку доставки
- **THEN** система логирует `registration.email.delivery_failed`
- **THEN** система возвращает `502 Bad Gateway` пользователю с возможностью повторной отправки через 60 секунд

---

## NON-FUNCTIONAL Requirements

### Requirement: Latency (ID: NFR-001)
Система SHALL обрабатывать 95-й перцентиль запросов к `/register` и `/verify-otp` не медленнее 300ms.

- **Измерение:** `http.server.request.duration{handler="/register", quantile=0.95}` ≤ 300ms
- **Измерение:** `http.server.request.duration{handler="/verify-otp", quantile=0.95}` ≤ 300ms

### Requirement: Availability (ID: NFR-002)
Система SHALL обеспечивать uptime endpoint'ов регистрации и верификации ≥ 99.9% в месяц.

- **Измерение:** `(total_requests - failed_requests) / total_requests` ≥ 0.999

### Requirement: Error rate (ID: NFR-003)
Система SHALL поддерживать error rate ≤ 0.5% для endpoint'ов регистрации и верификации.

- **Измерение:** `http.server.requests{status_code=~"5.."} / http.server.requests.total` ≤ 0.005

---

## Security Requirements

### Requirement: OTP brute-force protection (ID: NFR-SEC-001)
Система SHALL ограничивать количество попыток ввода OTP до 10 за 1 час на один user_id.

- **Покрывает:** TM-01
- **Метрика / verify:** `registration.otp.failed{reason="rate_limit"}` инкрементируется; тест TC-SEC-001

### Requirement: PII redaction in telemetry (ID: NFR-SEC-002)
Система SHALL обеспечивать redaction email в логах и спанах до экспорта в телеметрический пайплайн.

- **Покрывает:** TM-02
- **Метрика / verify:** 0 PII в логах/спанах; тест TC-SEC-002

### Requirement: Email enumeration protection (ID: NFR-SEC-003)
Система SHALL возвращать идентичные ответы для существующих и несуществующих email при регистрации.

- **Покрывает:** TM-03
- **Метрика / verify:** response time delta < 50ms между существующим и новым email; тест TC-SEC-003

---

## Observability Contract

> Заполняется для фичи с runtime-путём.
> Вход для `design.md` (точки инструментирования) и `test-specs` (assertions).

### Metrics

| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `http.server.request.duration` | metric (histogram) | каждый HTTP-запрос | `http.method`, `http.route`, `http.status_code`, `trace_id` | Prometheus | нет |
| `http.server.requests` | metric (counter) | каждый HTTP-запрос | `http.method`, `http.route`, `http.status_code` | Prometheus | нет |
| `registration.completed` | metric (counter) | успешная регистрация | `user.tenant_id`, `trace_id` | Prometheus | нет |
| `registration.otp.sent` | metric (counter) | отправка OTP | `channel=email`, `trace_id` | Prometheus | нет |
| `registration.otp.verified` | metric (counter) | успешная верификация | `trace_id` | Prometheus | нет |
| `registration.otp.failed` | metric (counter) | неудачная верификация | `reason="invalid"|"expired"|"rate_limit"`, `trace_id` | Prometheus | нет |
| `registration.consent.denied` | metric (counter) | отказ от согласия | `trace_id` | Prometheus | нет |
| `registration.email.delivery_failed` | metric (counter) | ошибка доставки email | `error_code`, `trace_id` | Prometheus | нет |

### Traces

| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `POST /register` | span (auto) | обработка регистрации | `http.method`, `http.route`, `trace_id` | Jaeger / OTel | нет |
| `POST /verify-otp` | span (auto) | обработка верификации | `http.method`, `http.route`, `trace_id` | Jaeger / OTel | нет |
| `otp.generate` | span (manual) | генерация OTP | `otp.ttl_seconds=300`, `trace_id` | Jaeger / OTel | нет |
| `otp.verify` | span (manual) | проверка OTP | `otp.success`, `otp.reason`, `trace_id` | Jaeger / OTel | нет |
| `email.send` | span (manual) | отправка OTP-email | `email.template="otp"`, `trace_id` | Jaeger / OTel | **email: redacted** |

### Logs

| Сигнал | Тип | Когда создаётся | Обязательные атрибуты | Куда | PII |
|--------|-----|-----------------|----------------------|------|-----|
| `security.rate_limit.triggered` | log (structured) | срабатывание rate-limit | `endpoint`, `limit_type`, `trace_id` | Loki | нет |
| `security.pii.redacted` | log (audit) | маскирование email | `field.name`, `original_length`, `trace_id` | Loki | **нет (redacted)** |
| `registration.failed` | log (structured) | ошибка регистрации | `error_code`, `http.status_code`, `trace_id` | Loki | **email: redacted** |

### SLO

| SLO | Целевое значение | Как измеряется |
|-----|------------------|----------------|
| Latency p95 | ≤ 300ms | `http.server.request.duration{quantile=0.95}` |
| Availability | ≥ 99.9% | `(requests - 5xx) / requests` |
| Error rate | ≤ 0.5% | `5xx / total requests` |
| OTP success rate | ≥ 85% | `registration.otp.verified / registration.otp.sent` |
