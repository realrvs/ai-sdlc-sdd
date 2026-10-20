# Traceability Matrix Template

> Этот файл используется для **инкрементального накопления** матрицы трассируемости.
> После каждого артефакта обновляйте соответствующую секцию.
> Полная матрица выводится в конце pipeline или по запросу.

---

## Accumulated Traceability Matrix

> **Обновляется после каждого артефакта.**
> Пустые ячейки = ещё не заполнены.

| ID | Тип | Spec § | TM § | ADR | API | Task IDs | Test IDs | Verify |
|----|-----|--------|------|-----|-----|----------|----------|--------|
| FR-001 | FR | `spec.md#req-001` | — | — | — | T-001, T-002 | TC-001 | — |
| FR-002 | FR | `spec.md#req-002` | — | — | — | T-003 | TC-002 | — |
| NFR-001 | NFR | `spec.md#nfr-001` | — | — | — | T-010 | TC-010 | SLO check |
| NFR-SEC-001 | NFR-SEC | `spec.md#nfr-sec-001` | TM-01 | ADR-001 | — | T-014 | TC-007 | Security test |
| NFR-OBS-001 | NFR-OBS | `spec.md#nfr-obs-001` | — | ADR-003 | — | T-019 | TC-001 (obs) | SLO + signal |
| TM-01 | TM | — | `threat-model.md#tm-01` | ADR-001 | — | T-014 | TC-007 | — |

---

## Pending Items

> Задачи, которые ещё не трассированы.

| Artifact | ID | Status | Blocked By |
|----------|----|--------|------------|
| spec.md | FR-003 | Draft | — |
| threat-model.md | TM-04 | Pending design review | — |
| tests.md | TC-008 | Not started | FR-003 |

---

## Rules

| ID | Spec § | TM § | ADR | API | Task IDs | Test IDs | Verify |
|----|--------|------|-----|-----|----------|----------|--------|
| FR | **непустая** | — | — | — | **непустая** | **непустая** | — |
| NFR | **непустая** | — | пусто* | пусто* | **непустая** | **непустая** | **непустая** |
| NFR-SEC | **непустая** | **непустая** | **непустая** | пусто* | **непустая** | **непустая** | **непустая** |
| NFR-OBS | **непустая** | — | **непустая** (если телеметрия) | — | **непустая** | **непустая** | **непустая** |
| TM-NNN | — | **непустая** | **непустая** | — | **непустая** | **непустая** | — |

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
