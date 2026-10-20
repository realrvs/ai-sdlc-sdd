# ADR-002: PII redaction strategy for telemetry pipeline

## Статус
Accepted

## Контекст
Threat Model выявил угрозу TM-02 (PII leak в телеметрию, Risk 12).
Email содержится в:
- HTTP request body (registration)
- Log messages (error logging, audit)
- OTel span attributes (email.send span)

Необходимо обеспечить redaction email до экспорта в Loki/Prometheus.

Связанный TM: TM-02
Связанный NFR-SEC: NFR-SEC-002

## Решение
Принята **Middleware-стратегия**: единый `PiiRedactionMiddleware` на уровне Auth Service.

**Механизм:**
1. Middleware перехватывает outgoing log records и span attributes
2. Regex-паттерн `[\w\.-]+@[\w\.-]+\.\w+` обнаруживает email
3. Email заменяется на `[REDACTED:<length>]` (например, `[REDACTED:18]`)
4. Audit-log `security.pii.redacted` создаётся при каждом обнаружении PII

**Конфигурация:**
```python
PII_PATTERNS = {
    "email": r"[\w\.-]+@[\w\.-]+\.\w+",
    "replacement": "[REDACTED:{length}]"
}
```

**Обоснование:**
1. Единая точка контроля — проще поддерживать и тестировать
2. Не требует изменений в каждом handler'е
3. Audit-log обеспечивает compliance-трейс для 152-ФЗ/GDPR

## Последствия

**Плюсы:**
- Единая точка управления PII-policy
- Audit-log для compliance
- Минимальное изменение existing code

**Минусы:**
- Performance overhead: regex на каждый log/span (~1-2ms)
- False positives: email-подобные паттерны в non-PII данных

**Риски:**
- **Риск:** Regex не покрывает все PII-паттерны (телефоны, паспортные данные) → **Митигация:** extensible pattern config; future ADR для расширенного PII-сканирования
- **Риск:** Performance degradation → **Митигация:** async regex + sampling для audit-log

**Влияние на наблюдаемость:**
- Добавлен сигнал `security.pii.redacted` (audit-log в Loki)
- Все логи/спаны проходят через redaction middleware
- `pii_redacted=true` flag в structured logs

## Альтернативы

| Альтернатива | Отклонена | Причина |
|---|---|---|
| Per-service redaction | да | Duplicated logic, hard to maintain consistency |
| Collector-level redaction | да | Too late — PII уже в application memory; harder to audit |
| No redaction (trust OTel) | да | Violates NFR-SEC-002 и 152-ФЗ requirements |
