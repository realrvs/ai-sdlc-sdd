# Test Specs: Auth — Registration & OTP

## Test Suite: auth

---

### TC-001: Successful registration with consent
- **Type**: Functional
- **Covers**: FR-001
- **Priority**: P0
#### Steps
1. Given clean database (no users)
2. When `POST /api/v1/register` with `{email: "new@example.com", consent: true}`
3. Then response is `201 Created` with `user_id` and `status: email_unverified`
4. Then OTP is stored in Redis with TTL 300
5. Then email with OTP is sent via mail-service
#### Data
- Input: `{ "email": "new@example.com", "consent": true }`
- Expected Output: `{ "user_id": "uuid", "status": "email_unverified" }`
#### Observability Assertions
- **Metrics:** `registration.completed` counter increased by 1
- **Logs:** no PII in Loki (email redacted)
- **Trace:** span `POST /register` exists with `trace_id`

---

### TC-002: Registration without consent
- **Type**: Functional
- **Covers**: FR-001, NFR-SEC-003
- **Priority**: P0
#### Steps
1. Given clean database
2. When `POST /api/v1/register` with `{email: "test@example.com", consent: false}`
3. Then response is `422 Unprocessable Entity` with `error_code: "consent_required"`
#### Data
- Input: `{ "email": "test@example.com", "consent": false }`
- Expected Output: `{ "error_code": "consent_required", "message": "User consent is required" }`
#### Observability Assertions
- **Metrics:** `registration.consent.denied` counter increased by 1
- **Logs:** `registration.failed` with `error_code: "consent_required"`

---

### TC-003: Duplicate email registration
- **Type**: Functional
- **Covers**: FR-001
- **Priority**: P0
#### Steps
1. Given user with email "existing@example.com" exists in database
2. When `POST /api/v1/register` with `{email: "existing@example.com", consent: true}`
3. Then response is `409 Conflict` with `error_code: "email_already_exists"`
#### Data
- Input: `{ "email": "existing@example.com", "consent": true }`
- Expected Output: `{ "error_code": "email_already_exists", "message": "Email is already registered" }`
#### Observability Assertions
- **Logs:** `registration.failed` with `error_code: "email_already_exists"`

---

### TC-004: Successful OTP verification
- **Type**: Functional
- **Covers**: FR-002
- **Priority**: P0
#### Steps
1. Given user with unverified email and valid OTP in Redis
2. When `POST /api/v1/verify-otp` with `{otp_code: "123456"}`
3. Then response is `200 OK` with `status: email_verified`
4. Then user status in DB is updated to `email_verified`
5. Then OTP is deleted from Redis
#### Data
- Input: `{ "otp_code": "123456" }`
- Expected Output: `{ "status": "email_verified" }`
#### Observability Assertions
- **Metrics:** `registration.otp.verified` counter increased by 1
- **Trace:** span `otp.verify` exists with `otp.success=true`
- **Logs:** no PII in Loki

---

### TC-005: Invalid OTP verification
- **Type**: Functional
- **Covers**: FR-002
- **Priority**: P1
#### Steps
1. Given user with unverified email
2. When `POST /api/v1/verify-otp` with `{otp_code: "000000"}`
3. Then response is `400 Bad Request` with `error_code: "invalid_otp"`
4. Then failed attempts counter is incremented
#### Data
- Input: `{ "otp_code": "000000" }`
- Expected Output: `{ "error_code": "invalid_otp", "message": "Invalid OTP code" }`
#### Observability Assertions
- **Metrics:** `registration.otp.failed{reason="invalid"}` counter increased by 1
- **Trace:** span `otp.verify` exists with `otp.success=false`

---

### TC-006: Expired OTP verification
- **Type**: Functional
- **Covers**: FR-002
- **Priority**: P1
#### Steps
1. Given user with expired OTP (TTL elapsed, > 5 min)
2. When `POST /api/v1/verify-otp` with the expired OTP code
3. Then response is `410 Gone` with `error_code: "otp_expired"`
#### Data
- Input: `{ "otp_code": "expired_code" }`
- Expected Output: `{ "error_code": "otp_expired", "message": "OTP code has expired" }`
#### Observability Assertions
- **Metrics:** `registration.otp.failed{reason="expired"}` counter increased by 1

---

### TC-007: Registration rate-limit (IP-based)
- **Type**: Security
- **Covers**: FR-003, NFR-SEC-001
- **TM Ref**: TM-01
- **Priority**: P0
#### Steps
1. Given clean rate-limit counters
2. When 4× `POST /api/v1/register` from same IP within 1 minute
3. Then 4th request returns `429 Too Many Requests` with `error_code: "rate_limit_exceeded"`
#### Data
- Input: 4× `{ "email": "ratelimit@example.com", "consent": true }`
- Expected Output: 4th → `{ "error_code": "rate_limit_exceeded" }`
#### Observability Assertions
- **Metrics:** `registration.otp.failed{reason="rate_limit"}` increased
- **Logs:** `security.rate_limit.triggered{endpoint="/register", limit_type="ip"}` in Loki
- **Security:** нет PII в логах (`pii_redacted=true`)

---

### TC-008: OTP brute-force protection
- **Type**: Security
- **Covers**: FR-003, NFR-SEC-001
- **TM Ref**: TM-01
- **Priority**: P0
#### Steps
1. Given user with unverified email and valid OTP
2. When 11× `POST /api/v1/verify-otp` with wrong OTP codes within 1 hour
3. Then 11th request returns `429 Too Many Requests` with `error_code: "account_temporarily_locked"`
4. Then account is locked for 30 minutes
#### Data
- Input: 11× `{ "otp_code": "wrong_code" }`
- Expected Output: 11th → `{ "error_code": "account_temporarily_locked" }`
#### Observability Assertions
- **Metrics:** `registration.otp.failed{reason="rate_limit"}` increased by 11
- **Logs:** `security.rate_limit.triggered{endpoint="/verify-otp", limit_type="user_id"}` in Loki
- **Security:** нет PII в логах

---

### TC-009: PII redaction in logs
- **Type**: Security
- **Covers**: NFR-SEC-002
- **TM Ref**: TM-02
- **Priority**: P0
#### Steps
1. Given PII redaction middleware is active
2. When `POST /api/v1/register` with email in request body
3. Then email is redacted in all log records
4. Then `security.pii.redacted` audit log is created
#### Data
- Input: `{ "email": "pii_test@example.com", "consent": true }`
- Expected: logs contain `[REDACTED:22]` instead of email
#### Observability Assertions
- **Logs:** Loki query `email="pii_test@example.com"` returns 0 results
- **Logs:** Loki query `security.pii.redacted` returns 1 result with `field.name="email"`
- **Security:** `pii_redacted=true` in audit log

---

### TC-010: Email enumeration protection
- **Type**: Security
- **Covers**: NFR-SEC-003
- **TM Ref**: TM-03
- **Priority**: P1
#### Steps
1. Given user "existing@example.com" exists in database
2. When `POST /api/v1/register` with existing email — measure response time
3. When `POST /api/v1/register` with new email — measure response time
4. Then response time delta < 50ms (identical response paths)
5. Then both return different status codes but same response structure
#### Data
- Existing: `{ "email": "existing@example.com", "consent": true }` → 409
- New: `{ "email": "new@example.com", "consent": true }` → 201 (or vice versa with timing mask)
- Expected: `|time_existing - time_new| < 50ms`
#### Observability Assertions
- **Metrics:** `http.server.request.duration` — no timing anomaly label
- **Logs:** no differentiation in log messages between existing/new email

---

### TC-011: OTP generation and storage
- **Type**: Functional
- **Covers**: FR-004
- **Priority**: P1
#### Steps
1. When user registers successfully
2. Then OTP is generated via `secrets.token_hex(3)` (6 hex digits)
3. Then OTP hash is stored in Redis with key `otp:<sha256_hash>`
4. Then Redis TTL is exactly 300 seconds
#### Data
- Expected: OTP = 6-char hex string, Redis key = `otp:<64-char-hash>`, TTL = 300
#### Observability Assertions
- **Metrics:** `registration.otp.sent` counter increased by 1
- **Trace:** span `otp.generate` exists with `otp.ttl_seconds=300`

---

### TC-012: Email delivery failure
- **Type**: Integration
- **Covers**: FR-004
- **Priority**: P1
#### Steps
1. Given user registers successfully, OTP generated and stored
2. When mail-service returns SMTP error (e.g., connection refused)
3. Then system logs `registration.email.delivery_failed`
4. Then user receives `502 Bad Gateway`
5. Then user can retry email resend after 60 seconds
#### Data
- Expected: `502` response, `registration.email.delivery_failed` log entry
#### Observability Assertions
- **Metrics:** `registration.email.delivery_failed` counter increased by 1
- **Logs:** structured log with `error_code` and `trace_id`

---

## Traceability: Tests → Requirements

| Test ID | Covers | Type | Priority |
|---------|--------|------|----------|
| TC-001 | FR-001 | Functional | P0 |
| TC-002 | FR-001, NFR-SEC-003 | Functional | P0 |
| TC-003 | FR-001 | Functional | P0 |
| TC-004 | FR-002 | Functional | P0 |
| TC-005 | FR-002 | Functional | P1 |
| TC-006 | FR-002 | Functional | P1 |
| TC-007 | FR-003, NFR-SEC-001 | Security | P0 |
| TC-008 | FR-003, NFR-SEC-001 | Security | P0 |
| TC-009 | NFR-SEC-002 | Security | P0 |
| TC-010 | NFR-SEC-003 | Security | P1 |
| TC-011 | FR-004 | Functional | P1 |
| TC-012 | FR-004 | Integration | P1 |
