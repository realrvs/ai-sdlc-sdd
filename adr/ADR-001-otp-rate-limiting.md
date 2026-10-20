# ADR-001: Rate-limiting strategy for registration and OTP endpoints

## Статус
Accepted

## Контекст
Threat Model выявил угрозу TM-01 (OTP brute-force, Risk 15).
Необходимо выбрать стратегию rate-limiting для защиты `/register` и `/verify-otp`.

Ключевые варианты:
- IP-based: ограничивает по IP-адресу источника
- User_id-based: ограничивает по user_id (после регистрации)
- Hybrid: IP для `/register`, user_id для `/verify-otp`

Связанный TM: TM-01
Связанный NFR-SEC: NFR-SEC-001

## Решение
Принята **Hybrid-стратегия**:
- `/register`: rate-limit по IP (3 запроса/мин, sliding window через Redis)
- `/verify-otp`: rate-limit по user_id (10 попыток/час, account lock 30 мин)

**Обоснование:**
1. `/register` — пользователь ещё не аутентифицирован, user_id неизвестен → IP-based единственный viable option
2. `/verify-otp` — user_id известен из OTP-связки → user_id-based точнее и не блокирует других пользователей при атаке на один аккаунт
3. Redis sliding window обеспечивает точность и distributed support

## Последствия

**Плюсы:**
- Защита от brute-force на обоих endpoint'ах
- Не блокирует легитимных пользователей при атаке на один аккаунт
- Redis sliding window точнее fixed-window

**Минусы:**
- IP может быть общим (NAT, прокси) → ложные срабатывания
- Account lock требует механизма разблокировки (auto-unlock через 30 мин)

**Риски:**
- **Риск:** IP-based rate-limit блокирует легитимных пользователей за общим NAT → **Митигация:** configurable threshold + alerting при high block rate

**Влияние на наблюдаемость:**
- Добавлен сигнал `security.rate_limit.triggered` (log)
- Добавлен счётчик `registration.otp.failed{reason="rate_limit"}` (metric)
- Prometheus alert `OtpBruteForceSpike` при >5 rate-limit событий/мин

## Альтернативы

| Альтернатива | Отклонена | Причина |
|---|---|---|
| Только IP-based | да | Не защищает `/verify-otp` от targeted brute-force |
| Только user_id-based | да | Не применим к `/register` (user_id неизвестен) |
| Fixed-window (1 мин) | да | Less accurate, allows burst at window boundary |
