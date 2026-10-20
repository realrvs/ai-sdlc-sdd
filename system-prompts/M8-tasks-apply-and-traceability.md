# M8: Tasks, Apply & Traceability

## Формат tasks.md

```markdown
## 001: Infrastructure — Redis and OTel
- [ ] T-001: Deploy Redis cluster for OTP storage (`infra/redis-deployment.yaml`)
  - **FR/NFR**: NFR-001, NFR-002
  - **Estimate**: 5
  - **Dependencies**: —
- [ ] T-002: Configure OTel Collector pipeline (`infra/otel-collector-config.yaml`)
  - **FR/NFR**: NFR-OBS-001
  - **Estimate**: 3
  - **Dependencies**: T-001
```

### Правила задач
- **Атомарность**: одна задача = один commit
- **ID**: `T-NNN`, NNN — 3 цифры, группировка по фазам (001-099, 101-199…)
- **Связи**: каждая задача обязана ссылаться на ≥1 FR/NFR/ADR
  (для security-задач — NFR-SEC + TM; **для observability-задач — NFR-OBS + сигнал**)
- **Оценка**: story-points (1, 2, 3, 5, 8) или часы
  - **Шкала SP**: 1 = ≤4ч, 2 = ≤1день, 3 = ≤2дня, 5 = 3+ дней
  - **Шкала Hours**: 2, 4, 8, 16, 24, 40
- **Зависимости**: только на другие T-NNN из этого файла
- **Задачи-сироты запрещены**: нет ссылки на FR/NFR/ADR = задача не допустима
- **Observability-задачи**:
  - OTel SDK + auto-instrumentation
  - ручные спаны для бизнес-операций
  - настройка экспорта в Collector
  - алерты в Prometheus rules
  - dashboard в Grafana
  - redaction-конфигурация (если TM выявил PII-угрозы)

---

## Формат apply.md

```markdown
## Миграции
- [ ] <migration-name>: <description> (<file-path>)
- [ ] Rollback: <rollback-steps>

## Feature Flags
| Flag | Default | TTL | Связанный FR | Связанный TM |
|---|---|---|---|---|

## Canary Deployment
1. Deploy canary 5% трафика
2. Monitor SLO canary vs baseline:
   - p95 latency ≤ <target>ms
   - error rate ≤ <target>%
   - business KPI ≥ <target>%
3. **Auto-rollback при degradation > 10% по любому SLO**
   - Trigger: Prometheus alert `SloDegradation` fires
   - Action: feature flag toggle → ротация pods → rollback deployment
4. Если canary stable 24h → 50% → 100%
5. Отключить feature flag для old code path

## Observability Verification
> Проверка, что сигналы из Observability Contract реально видны в бэкендах.
- [ ] В Grafana виден дашборд с RED-метриками для фичи
- [ ] В Loki есть логи с `trace_id` для тестового запроса
- [ ] В Jaeger/OTel виден трейс, связанный с `trace_id` из HTTP
- [ ] Prometheus: алерты срабатывают на тестовом сценарии
- [ ] `security.pii.redacted` audit-лог фиксируется при redaction
- [ ] Retention policy соответствует NFR-SEC-NNN

## Rollback Plan
1. Отключить feature flag
2. Ротация pods для удаления нового code path
3. Откат миграций (если применены)
4. Проверка, что old flow работает без regressions
5. Проверка, что telemetry pipeline не содержит аномалий
```

---

## Traceability Matrix

> **Накапливается инкрементально в `context.md`** после каждого артефакта.
> Полная матрица выводится в конце (после `apply`) или по запросу.

| ID | Тип | Spec § | TM § | ADR | API Endpoint | Task IDs | Test IDs | Verify Method |
|----|-----|--------|------|-----|-------------|----------|----------|---------------|

### Правила заполнения
| ID | Spec § | TM § | ADR | API | Task IDs | Test IDs | Verify Method |
|----|--------|------|-----|-----|----------|----------|---------------|
| FR | **непустая** | — | — | — | **непустая** | **непустая** | — |
| NFR | **непустая** | — | пусто* | пусто* | **непустая** | **непустая** | **непустая** |
| NFR-SEC | **непустая** | **непустая** | **непустая** | пусто* | **непустая** | **непустая** (Security) | **непустая** |
| NFR-OBS | **непустая** | — | **непустая** (если затрагивает телеметрию) | — | **непустая** | **непустая** (Observability) | **непустая** (SLO + сигнал) |
| TM-NNN | — | **непустая** | **непустая** | — | **непустая** | **непустая** (Security) | — |

> *API пустое, если NFR не относится к API (напр. latency, availability)

### Запреты
- ❌ Задачи-сироты (без ссылки на FR/NFR/ADR)
- ❌ FR без Spec § / Task / Test
- ❌ NFR без Verify Method
- ❌ NFR-SEC без TM Ref
- ❌ TM-NNN без контроля и без связанного NFR-SEC
- ❌ ADR по безопасности без ссылки на TM-NNN
- ❌ NFR-OBS без сигнала в Observability Contract
- ❌ NFR-OBS без Observability Assertion в тесте
- ❌ Сигнал с PII без redaction-контроля
