# Proposal: User Registration with OTP

## Why
Пользователи регистрируются в системе по email, но текущий процесс не включает верификацию. Это приводит к:
- Дубликатам аккаунтов
- Мошеннической регистрации (боты)
- Невозможности восстановить аккаунт по email

Бизнес-контекст: рост регистраций на 40% QoQ требует автоматизированной верификации с защитой от brute-force.

**Метрики:**
- Текущий drop-off на регистрации: 35% (без OTP)
- Целевой drop-off: ≤ 25% (с OTP)
- Доля бот-регистраций: 12% → ≤ 1%

## What Changes
- Новый endpoint `/api/v1/register` — создание пользователя с отправкой OTP
- Новый endpoint `/api/v1/verify-otp` — верификация OTP-кода
- Rate-limit на оба endpoint'а (3 попытки/мин, 10 попыток/час на IP)
- Consent-проверка перед регистрацией (обязательное согласие на обработку ПДн)
- OTP: 6-значный код, TTL 5 минут, однократное использование
- Email-уведомления с OTP через существующий mail-service

## Impact
- **Затронутые capabilities:**
  - `user.registration` (новая)
  - `user.verification` (новая)
  - `user.consent` (расширение)
  - `notification.email` (расширение)
- **Затронутые команды:**
  - `auth-team` — backend, endpoints, OTP-логика
  - `platform-team` — mail-service integration, rate-limiting middleware
  - `compliance-team` — consent-проверка, 152-ФЗ соответствие
- **Целевые метрики:**
  - Latency p95 ≤ 300ms на `/register` и `/verify-otp`
  - Availability ≥ 99.9%
  - Error rate ≤ 0.5%
  - OTP success rate ≥ 85%
- **Security-профиль:**
  - Regulatory: 152-ФЗ, GDPR
  - Assets: PII (email), secrets (OTP-токены)
  - Методология Threat Modeling: LINDDUN + STRIDE
- **Operability-профиль:**
  - Стек: OpenTelemetry, Prometheus, Loki, Grafana
  - SLO: latency p95 ≤ 300ms, availability ≥ 99.9%, error rate ≤ 0.5%
  - Existing instrumentation: otel-sdk-python

## Operability Promise
> Как мы узнаем, что фича работает в production?

- **SLO:**
  - Latency p95 ≤ 300ms (HTTP-запросы к `/register`, `/verify-otp`)
  - Availability ≥ 99.9% (uptime endpoint'ов)
  - Error rate ≤ 0.5% (5xx + 4xx rate-limit)
- **Ключевые сигналы:**
  - RED-метрики: `requests`, `errors`, `duration` на `/register`, `/verify-otp`
  - Бизнес-метрики: `registration.completed`, `registration.otp.sent`, `registration.otp.verified`, `registration.consent.denied`
  - Критичные логи: `security.rate_limit.triggered`, `security.pii.redacted`, `registration.failed`
- **Что НЕ покрыто:**
  - UX-метрики (drop-off на шаге OTP, время до ввода кода)
  - Email deliverability (bounce, spam-фильтры)
  - A/B-тестирование UX OTP-формы
