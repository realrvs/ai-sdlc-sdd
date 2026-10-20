# ADR-003: OpenTelemetry auto-instrumentation for FastAPI

## Статус
Accepted

## Контекст
Observability Contract определяет 8 metrics, 3 spans и 3 logs для фичи.
Необходимо выбрать подход к instrumentации:
- Full manual (каждый span/metric создаётся вручную)
- Auto-instrumentation (OTel SDK auto-instruments HTTP, DB, Redis)
- Hybrid (auto для HTTP/DB, manual для бизнес-спанов)

Связанный NFR-OBS: NFR-OBS-001, NFR-OBS-002

## Решение
Принята **Hybrid-стратегия**:
- **Auto-instrumentation:** HTTP requests (FastAPI middleware), DB queries (psycopg2), Redis (redis-py)
- **Manual spans:** `otp.generate`, `otp.verify`, `email.send` — бизнес-операции
- **Manual metrics:** `registration.*`, `security.*` — бизнес-метрики
- **Manual logs:** `security.*` — security audit-логи

**Обоснование:**
1. Auto-instrumentation покрывает 70% Observability Contract (HTTP metrics, DB spans) без кода
2. Manual spans нужны для бизнес-операций (OTP lifecycle), которые auto не понимает
3. Manual metrics нужны для бизнес-KPI (registration.completed, otp.verified)
4. otel-sdk-python уже в проекте — zero additional dependency

## Последствия

**Плюсы:**
- Минимальный boilerplate code
- HTTP metrics и spans из коробки
- Easy to add manual spans for business logic

**Минусы:**
- Auto-instrumentation не создаёт бизнес-спаны (otp.generate и т.д.)
- Manual instrumentation требует дисциплины (developers must remember to add spans)

**Риски:**
- **Риск:** Developers skip manual spans → gaps in observability → **Митигация:** lint rule + code review checklist
- **Риск:** Auto-instrumentation adds unwanted attributes → **Митигация:** OTel attribute filters in Collector

**Влияние на наблюдаемость:**
- Auto: `http.server.request.duration`, `http.server.requests`, DB spans
- Manual: `otp.generate`, `otp.verify`, `email.send` spans
- Manual: `registration.*` counters, `security.*` logs

## Альтернативы

| Альтернатива | Отклонена | Причина |
|---|---|---|
| Full manual | да | Excessive boilerplate, inconsistent instrumentation |
| Full auto | да | Cannot capture business logic (OTP lifecycle, consent) |
