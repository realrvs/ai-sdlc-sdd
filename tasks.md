# Tasks: User Registration with OTP

## 001: Infrastructure — Redis and OTel
- [ ] T-001: Deploy Redis cluster for OTP storage and rate-limiting (`infra/redis-deployment.yaml`)
  - **FR/NFR**: NFR-001, NFR-002
  - **Estimate**: 5
  - **Dependencies**: —
- [ ] T-002: Configure OTel Collector pipeline for auth service (`infra/otel-collector-config.yaml`)
  - **FR/NFR**: NFR-OBS-001
  - **TM Ref**: —
  - **Estimate**: 3
  - **Dependencies**: T-001
- [ ] T-003: Add Prometheus alert rules for OTP brute-force (`infra/prometheus-rules.yaml`)
  - **FR/NFR**: NFR-OBS-002, NFR-SEC-001
  - **TM Ref**: TM-01
  - **Estimate**: 2
  - **Dependencies**: T-002

## 002: Core — Registration endpoint
- [ ] T-004: Implement `POST /api/v1/register` endpoint (`src/auth/routes/register.py`)
  - **FR/NFR**: FR-001
  - **Estimate**: 3
  - **Dependencies**: —
- [ ] T-005: Implement consent validation middleware (`src/auth/middleware/consent.py`)
  - **FR/NFR**: FR-001, NFR-SEC-003
  - **Estimate**: 2
  - **Dependencies**: T-004
- [ ] T-006: Implement duplicate email check with timing mask (`src/auth/services/user_service.py`)
  - **FR/NFR**: FR-001, NFR-SEC-003
  - **TM Ref**: TM-03
  - **Estimate**: 2
  - **Dependencies**: T-004
- [ ] T-007: Add DB migration for users table (`migrations/V001__create_users.sql`)
  - **FR/NFR**: FR-001
  - **Estimate**: 2
  - **Dependencies**: —

## 003: OTP — Generation and storage
- [ ] T-008: Implement OTP generation service (`src/auth/services/otp_service.py`)
  - **FR/NFR**: FR-004
  - **Estimate**: 2
  - **Dependencies**: —
- [ ] T-009: Implement OTP Redis storage with TTL (`src/auth/repositories/otp_repo.py`)
  - **FR/NFR**: FR-004
  - **Estimate**: 2
  - **Dependencies**: T-001
- [ ] T-010: Add manual OTel span for `otp.generate` (`src/auth/services/otp_service.py`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 1
  - **Dependencies**: T-008

## 004: OTP — Verification
- [ ] T-011: Implement `POST /api/v1/verify-otp` endpoint (`src/auth/routes/verify_otp.py`)
  - **FR/NFR**: FR-002
  - **Estimate**: 3
  - **Dependencies**: T-008
- [ ] T-012: Implement OTP verification logic (hash comparison, DB update) (`src/auth/services/otp_service.py`)
  - **FR/NFR**: FR-002
  - **Estimate**: 2
  - **Dependencies**: T-009
- [ ] T-013: Add manual OTel span for `otp.verify` (`src/auth/services/otp_service.py`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 1
  - **Dependencies**: T-012

## 005: Security — Rate limiting
- [ ] T-014: Implement IP-based rate-limit middleware for `/register` (`src/auth/middleware/rate_limit.py`)
  - **FR/NFR**: FR-003, NFR-SEC-001
  - **TM Ref**: TM-01
  - **Estimate**: 3
  - **Dependencies**: T-001
- [ ] T-015: Implement user_id-based rate-limit for `/verify-otp` with account lock (`src/auth/middleware/rate_limit.py`)
  - **FR/NFR**: FR-003, NFR-SEC-001
  - **TM Ref**: TM-01
  - **Estimate**: 3
  - **Dependencies**: T-014
- [ ] T-016: Add `security.rate_limit.triggered` log signal (`src/auth/middleware/rate_limit.py`)
  - **FR/NFR**: NFR-OBS-002
  - **TM Ref**: TM-01
  - **Estimate**: 1
  - **Dependencies**: T-015

## 006: Security — PII redaction
- [ ] T-017: Implement PII redaction middleware (`src/auth/middleware/pii_redaction.py`)
  - **FR/NFR**: NFR-SEC-002
  - **TM Ref**: TM-02
  - **Estimate**: 3
  - **Dependencies**: —
- [ ] T-018: Add `security.pii.redacted` audit log signal (`src/auth/middleware/pii_redaction.py`)
  - **FR/NFR**: NFR-SEC-002, NFR-OBS-002
  - **TM Ref**: TM-02
  - **Estimate**: 1
  - **Dependencies**: T-017

## 007: Observability — Metrics and traces
- [ ] T-019: Add business metrics (registration.completed, otp.sent, otp.verified) (`src/auth/services/metrics.py`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 2
  - **Dependencies**: T-004
- [ ] T-020: Add manual OTel spans for `email.send` (`src/auth/services/mail_client.py`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 1
  - **Dependencies**: T-004
- [ ] T-021: Configure W3C traceparent propagation in HTTP middleware (`src/auth/middleware/trace.py`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 2
  - **Dependencies**: T-002

## 008: Integration — Email delivery
- [ ] T-022: Implement mail-service client with OTP template (`src/auth/services/mail_client.py`)
  - **FR/NFR**: FR-004
  - **Estimate**: 3
  - **Dependencies**: —
- [ ] T-023: Handle email delivery failure with retry logic (`src/auth/services/mail_client.py`)
  - **FR/NFR**: FR-004
  - **Estimate**: 2
  - **Dependencies**: T-022
- [ ] T-024: Add `registration.email.delivery_failed` metric and log (`src/auth/services/mail_client.py`)
  - **FR/NFR**: NFR-OBS-002
  - **Estimate**: 1
  - **Dependencies**: T-023

## 009: Testing
- [ ] T-025: Write functional tests for registration flow (`tests/test_register.py`)
  - **FR/NFR**: FR-001, FR-002
  - **Test IDs**: TC-001, TC-002, TC-003, TC-004, TC-005, TC-006
  - **Estimate**: 5
  - **Dependencies**: T-004, T-011
- [ ] T-026: Write security tests for rate-limiting (`tests/test_rate_limit.py`)
  - **FR/NFR**: FR-003, NFR-SEC-001
  - **TM Ref**: TM-01
  - **Test IDs**: TC-007, TC-008
  - **Estimate**: 3
  - **Dependencies**: T-015
- [ ] T-027: Write security tests for PII redaction (`tests/test_pii_redaction.py`)
  - **FR/NFR**: NFR-SEC-002
  - **TM Ref**: TM-02
  - **Test IDs**: TC-009
  - **Estimate**: 2
  - **Dependencies**: T-017
- [ ] T-028: Write security tests for email enumeration (`tests/test_enumeration.py`)
  - **FR/NFR**: NFR-SEC-003
  - **TM Ref**: TM-03
  - **Test IDs**: TC-010
  - **Estimate**: 2
  - **Dependencies**: T-006
- [ ] T-029: Write integration tests for email delivery (`tests/test_email_delivery.py`)
  - **FR/NFR**: FR-004
  - **Test IDs**: TC-011, TC-012
  - **Estimate**: 3
  - **Dependencies**: T-022
- [ ] T-030: Write observability assertion tests (`tests/test_observability.py`)
  - **FR/NFR**: NFR-OBS-001, NFR-OBS-002
  - **Test IDs**: TC-001 (assertions), TC-004 (assertions), TC-009 (assertions)
  - **Estimate**: 3
  - **Dependencies**: T-019, T-021

## 010: Deployment and verification
- [ ] T-031: Create Grafana dashboard for registration metrics (`dashboards/auth-registration.json`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 3
  - **Dependencies**: T-019
- [ ] T-032: Implement canary deployment config (`deploy/canary-auth.yaml`)
  - **FR/NFR**: NFR-002
  - **Estimate**: 2
  - **Dependencies**: T-001
- [ ] T-033: Run observability verification checklist (`apply.md` § Observability Verification)
  - **FR/NFR**: NFR-OBS-001, NFR-OBS-002
  - **Estimate**: 2
  - **Dependencies**: T-031, T-032
