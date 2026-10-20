# Apply: User Registration with OTP

## Миграции
- [ ] V001__create_users: Создать таблицу users с полями email, status, consent_record, created_at (`migrations/V001__create_users.sql`)
  - Rollback: `DROP TABLE users`
- [ ] V002__create_user_consent_audit: Создать таблицу audit для consent-записей (`migrations/V002__create_consent_audit.sql`)
  - Rollback: `DROP TABLE user_consent_audit`

## Feature Flags

| Flag | Default | TTL | Связанный FR | Связанный TM |
|---|---|---|---|---|
| `feature.registration.otp.enabled` | false | 30 дней | FR-001, FR-002 | — |
| `feature.registration.otp.rate_limit.strict` | false | 14 дней | FR-003 | TM-01 |
| `feature.registration.pii_redaction.enabled` | true | — | NFR-SEC-002 | TM-02 |

## Canary Deployment
1. Deploy canary 5% трафика на новый endpoint `/api/v1/register`
2. Monitor SLO canary vs baseline:
   - p95 latency ≤ 300ms
   - error rate ≤ 0.5%
   - OTP success rate ≥ 85%
3. Auto-rollback при degradation > 10% по любому SLO
4. Feature flag toggle → мгновенная ротация pods (kill old, start new)
5. Если canary stable 24h → увеличить до 50%
6. Если 50% stable 24h → увеличить до 100%
7. Отключить feature flag для old code path

## Observability Verification
> Проверка, что сигналы из Observability Contract реально видны в бэкендах.

- [ ] В Grafana виден дашборд с RED-метриками для `/register`, `/verify-otp`
- [ ] В Loki есть логи с `trace_id` для тестового запроса (TC-001)
- [ ] В Jaeger/OTel виден LLM-трейс (span `POST /register`) с `trace_id` из W3C Trace Context
- [ ] Prometheus: алерт `OtpBruteForceSpike` срабатывает на тестовом сценарии (TC-008)
- [ ] `security.pii.redacted` audit-лог фиксируется при регистрации с email
- [ ] `registration.otp.failed{reason="rate_limit"}` инкрементируется при rate-limit (TC-007)
- [ ] В Loki нет PII (email) в логах — query `email="test@example.com"` возвращает 0

## Rollback Plan
1. Отключить feature flag `feature.registration.otp.enabled`
2. Ротация pods для удаления нового code path
3. Откат миграций (если применены):
   - `DROP TABLE user_consent_audit`
   - `DROP TABLE users` (если таблица новая)
4. Проверка, что old registration flow работает без OTP
5. Проверка, что telemetry pipeline не содержит аномалий после отката
