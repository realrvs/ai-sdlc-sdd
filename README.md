![Status](https://img.shields.io/badge/status-production--ready-green)
# AI-SDLC-SDD: Spec-Driven Development с Threat Modeling и Observability by Design

Репозиторий проекта **AI-SDLC-SDD**, посвященного методологии спецификационно-ориентированной разработки (Spec-Driven Development, SDD) с использованием искусственного интеллекта.

Проект предоставляет системный промпт для AI-ассистента (роль: системный аналитик и архитектор), который помогает генерировать полный пакет проектной документации по стандарту **OpenSpec v1.x**, интегрируя:
- Обязательный этап **Threat Modeling** с автоматическим выбором методологии
- **Observability by Design** — наблюдаемость как артефакт проектирования (не постфактум)
- Полную трассируемость от требований до задач и тестов

---

## 🎯 Назначение

Файл `openspec-prompt.md` содержит жесткий регламент (пайплайн) для LLM, который позволяет:

- Генерировать артефакты строго последовательно: `proposal → specs → design → threat-model → adr → api-contract → test-specs → tasks → apply`
- Автоматически выбирать методологию Threat Modeling (STRIDE, OWASP Top 10 for LLM, LINDDUN и др.) в зависимости от `security_context`
- Формировать **Observability Contract** на этапе спецификаций — каждый runtime-путь обязан иметь метрики, логи и трейсы
- Формировать матрицу трассируемости (Traceability Matrix) и проверять артефакты на соответствие Definition of Done

---

## 📂 Структура репозитория

| Файл / Папка | Описание |
|---|---|
| `openspec-prompt.md` | Основной системный промпт. Роль, входные параметры, правила генерации каждого артефакта |
| `proposal.md` | Бизнес-контекст, изменения, Impact, **Operability Promise** (SLO, ключевые сигналы) |
| `specs/{domain}/spec.md` | FR/NFR/NFR-SEC + **Observability Contract** (метрики, спаны, логи, SLO) |
| `design.md` | Компонентная архитектура, sequence-диаграммы, точки инструментирования, корреляция |
| `threat-model.md` | Выбранная методология, Trust Boundaries, матрица угроз, влияние на ADR и наблюдаемость |
| `adr/ADR-NNN-*.md` | Архитектурные решения со ссылкой на TM и NFR-OBS |
| `api-contract.yaml` | OpenAPI 3.1.0 с `traceparent`, `X-Request-ID`, `ErrorResponse.trace_id` |
| `test-specs/{domain}/tests.md` | Тесты с **Observability Assertions** (спаны, метрики, логи, PII) |
| `tasks.md` | Атомарные задачи (1 задача = 1 commit), связанные с FR/NFR/TM |
| `apply.md` | Миграции, feature flags, canary deployment, **Observability Verification** |
| `context.md` | **Shared state** — обновляется после каждого артефакта, содержит накопительную Traceability Matrix. Единственный источник истины между модулями M1–M8. |

---

## 🚀 Как использовать

### 1. Подготовка
Откройте диалог с LLM (ChatGPT, Claude, YandexGPT и т.д.), которая поддерживает длинный контекст и генерацию Markdown/Mermaid.

### 2. Загрузка промпта
Передайте модели содержимое файла `openspec-prompt.md` в качестве системной инструкции или первого сообщения.

### 3. Ввод данных
Модель запросит обязательные параметры:

```yaml
feature: "User Registration with OTP"
description: "Email-based registration с верификацией через OTP, rate-limit и consent-проверкой"
domain: auth
constraints: ["PostgreSQL", "REST", "OAuth2", "152-ФЗ"]
security_context:
  regulatory: ["152-ФЗ", "GDPR"]
  ai: []
  assets: ["PII", "secrets"]
observability_context:
  stack: ["otel", "loki", "prometheus", "grafana"]
  slo:
    latency_p95_ms: 300
    availability: "99.9%"
    error_rate: "0.5%"
  existing_instrumentation: ["otel-sdk-python"]
```

### 4. Генерация
Модель начнет пошаговую генерацию артефактов. После каждого шага — checkpoint с вопросом `Продолжить? (y/n)`.

### 5. Результат
Набор файлов (`.md`, `.yaml`), готовых к сохранению в репозиторий проекта.

---

## 🛡️ Threat Modeling

Промпт содержит встроенную логику выбора методологии анализа угроз на основе `security_context`:

| `security_context` | Методология |
|---|---|
| `ai: ["RAG", "agent"]` + PII | **STRIDE + OWASP Top 10 for LLM** |
| `ai: ["RAG", "agent"]`, без PII | **OWASP Top 10 for LLM** |
| `regulatory: ["GDPR", "152-ФЗ"]` | **LINDDUN** (+ STRIDE при наличии auth) |
| Классическая система без AI | **STRIDE** |
| Внутренняя фича, нет чувствительных данных | **Baseline** (минимальный) |

**Ключевые особенности:**
- Этап `threat-model` вставляется **между** `design` и `adr` — результаты влияют на архитектурные решения
- Пропуск этапа заблокирован (кроме минимального viable TM)
- Каждая угроза `TM-NNN` связывается с NFR-SEC и ADR
- Telemetry pipeline рассматривается как актив и вектор атаки (PII в логах, poisoning)

---

## 🔭 Observability by Design

Наблюдаемость — не постфактум DevOps, а артефакт проектирования.

| Шаг пайплайна | Вклад в наблюдаемость |
|---|---|
| `proposal` | **Operability Promise**: SLO, ключевые сигналы, что НЕ покрыто |
| `specs` | **Observability Contract**: метрики, спаны, логи, SLO с измеримыми критериями |
| `design` | **Точки инструментирования**: где создаются сигналы, проброс `traceparent` |
| `threat-model` | **Redaction/PII**: защита телеметрии, TB для telemetry pipeline |
| `test-specs` | **Observability Assertions**: проверка спанов, метрик, логов, отсутствия PII |
| `tasks` | Задачи на OTel SDK, ручные спаны, алерты, дашборды, redaction-конфиг |
| `apply` | **Observability Verification**: чеклист проверки сигналов в production |

---

## 📊 Пример: User Registration with OTP

В репозитории представлен полный набор артефактов для фичи **User Registration with OTP** (домен `auth`):

| Артефакт | Содержание |
|---|---|
| `proposal.md` | Operability Promise: SLO p95 ≤ 300ms, availability ≥ 99.9%, error rate ≤ 0.5% |
| `specs/auth/spec.md` | 4 FR, 3 NFR, 3 NFR-SEC, Observability Contract (8 метрик, 3 спана, 3 лога) |
| `design.md` | Sequence-диаграмма, точки инструментирования, W3C Trace Context |
| `threat-model.md` | LINDDUN + STRIDE, 6 угроз (TM-01…TM-06), Trust Boundaries |
| `adr/` | 3 ADR: rate-limiting, PII redaction, OTel auto-instrumentation |
| `api-contract.yaml` | 2 endpoint'а, `trace_id` в ErrorResponse, `traceparent` в параметрах |
| `test-specs/auth/tests.md` | 12 тестов (6 security), Observability Assertions |
| `tasks.md` | 33 атомарные задачи, 5 фаз |
| `apply.md` | Canary deployment, feature flags, Observability Verification |

---

## 📑 Поддержка ГОСТ (Опционально)

Если при инициализации указать `gost_compliance: true`, ассистент будет параллельно с OpenSpec-артефактами генерировать документы по ГОСТ 34/19 (например, ТЗ по ГОСТ 34.602-89 или ПМИ по ГОСТ 34.603-92), сохраняя трассируемость между ними.

---

## 🧩 Модульность (M1–M8)

Для моделей с ограниченным контекстом промпт разбит на модульные файлы в `system-prompts/`:

| Модуль | Файл | Содержимое |
|--------|------|------------|
| **M1** | `M1-role-and-pipeline.md` | Роль, pipeline, входные данные, error recovery |
| **M2** | `M2-proposal.md` | Формат proposal.md + Operability Promise |
| **M3** | `M3-specs-and-observability.md` | Specs + Observability Contract + dynamic limits |
| **M4** | `M4-design.md` | Design + точки инструментирования + correlation |
| **M5** | `M5-threat-model.md` | Threat Modeling + risk scoring (L×I) + Top-3 |
| **M6** | `M6-adrs.md` | ADR format + security/observability links |
| **M7** | `M7-api-and-tests.md` | API Contract + Test Specs + Observability Assertions |
| **M8** | `M8-tasks-apply-and-traceability.md` | Tasks + Apply + Traceability Matrix |

**Как использовать модули:**
1. Загрузите M1 как базовый системный промпт
2. По мере прохождения pipeline подгружайте соответствующий модуль (M2 → M3 → …)
3. Это экономит контекстное окно и ускоряет генерацию

---

## 📐 Dynamic Limits (по домену)

Лимиты FR/NFR адаптируются под домен фичи:

| Domain | FR range | NFR range | NFR-OBS range | NFR-SEC range |
|--------|----------|-----------|---------------|---------------|
| `auth` | 5–15 | 3–8 | 2–5 | 2–5 |
| `api` | 10–25 | 5–10 | 3–6 | 3–8 |
| `data-pipeline` | 15–40 | 5–12 | 3–8 | 2–6 |
| `workflow` | 10–30 | 5–10 | 3–6 | 2–5 |
| `ui` | 5–20 | 3–8 | 1–4 | 1–3 |
| `infra` | 5–15 | 5–10 | 2–5 | 1–4 |
| **default** | **5–15** | **3–8** | **2–5** | **1–5** |

> Если диапазон превышен — фича разделяется на подфичи с новым domain.

---

## 🎯 Risk Scoring в Threat Modeling

Каждая угроза оценивается по формуле **Risk = Likelihood (1–5) × Impact (1–5)**.

| Score | Level | Action |
|-------|-------|--------|
| 20–25 | Critical | Immediate mitigation, block release |
| 12–19 | High | Mitigation в текущем спринте |
| 6–11 | Medium | Mitigation в следующем спринте |
| 1–5 | Low | Backlog, monitor |

**Top-3** угрозы с наивысшим Risk выделяются для immediate mitigation и явно указываются в threat-model.md.

---

## 🚀 Canary Deployment с Auto-Rollback

Canary deployment включает **автоматический rollback** при degradation SLO:

1. Deploy canary 5% трафика
2. Monitor SLO canary vs baseline (p95 latency, error rate, business KPI)
3. **Auto-rollback при degradation > 10%** по любому SLO:
   - **Trigger:** Prometheus alert `SloDegradation` fires
   - **Action:** feature flag toggle → ротация pods → rollback deployment
4. Если canary stable 24h → 50% → 100%
5. Отключить feature flag для old code path

---

## 📄 Шаблоны

В папке `templates/` доступны переиспользуемые шаблоны для других проектов:

| Шаблон | Описание |
|--------|----------|
| `observability-contract.template.md` | Observability Contract (метрики, спаны, логи, SLO) |
| `threat-matrix.template.md` | Threat Matrix с Risk Scoring (L×I) |
| `traceability-matrix.template.md` | Traceability Matrix с инкрементальным накоплением |

Используйте их как основу для новых проектов без копирования всего промпта.

---

## 🔄 Error Recovery

Pipeline имеет встроенную обработку ошибок. Вот типичные сценарии:

### SLO не заданы
> **Сигнал:** `observability_context.slo` пуст
> **Действие:** предложить baseline — p95 ≤ 500ms, availability ≥ 99%, error rate ≤ 1%
> **Результат:** пользователь подтверждает или задаёт свои значения

### Превышен лимит FR/NFR
> **Сигнал:** 25 FR для домена `auth` (лимит 5–15)
> **Действие:** предложить разделить на подфичи с новым domain
> **Результат:** `auth-registration` + `auth-verification`

### Threat Model не находит угроз
> **Сигнал:** TM-матрица пуста
> **Действие:** перепроверить `security_context`; при подтверждении — пометить `TM: no-relevant-threats` и продолжить

### Конфликт TM vs Observability Contract
> **Сигнал:** TM выявил PII в поле `user.email` в логе, но Observability Contract не предусматривает redaction
> **Действие:** остановить pipeline, предложить 3 варианта:
> 1. Добавить NFR-OBS на redaction (рекомендуется)
> 2. Убрать поле из логов на уровне кода
> 3. Повысить уровень шифрования телеметрии
> **Результат:** пользователь выбирает вариант, pipeline продолжается

### Отказ от артефакта
> **Первый отказ:** задать 1 уточняющий вопрос, предложить 2 варианта
> **Второй отказ на том же артефакте:** предложить упрощённую версию (simplified)
> **Отказ на 2+ артефактах подряд:** перейти к `tasks` с пометкой `⚠️ skipped: <names>`
> **Отказ на threat-model:** нельзя пропустить — minimum viable TM (Trust Boundaries + Top-3)

---

## 📋 Shared State: context.md

Файл `context.md` — **единственный shared state** между модулями M1–M8.

**Как работает:**
1. После каждого артефакта AI обновляет `context.md`
2. Содержимое передаётся как входные данные в следующий модуль
3. **Traceability Matrix накапливается инкрементально** — после каждого шага добавляются новые строки
4. В конце pipeline полная матрица выводится или запрашивается отдельно

**Структура:**
```markdown
# Context
## Feature: ...
## Threat Model: методология + ключевые угрозы
## Принятые ADR: список с TM-ссылками
## Observability: стек, SLO, ключевые сигналы
## Traceability Matrix (INCREMENTAL):
| ID | Тип | Spec § | TM § | ADR | API | Task IDs | Test IDs | Verify |
...
```

> **Важно:** `context.md` подаётся как входные данные в каждый следующий модуль.
> Цитируйте только релевантные секции, чтобы не переполнять контекстное окно.

---

## 🔍 CI-проверка Traceability Matrix

Для production-использования добавьте GitHub Action, которая проверяет DoD после каждого коммита:

```yaml
# .github/workflows/traceability-check.yml
name: Traceability Check
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check orphan tasks
        run: |
          # Проверка: все T-NNN в tasks.md ссылаются на FR/NFR/ADR
          grep -Pzo '(?s)T-\d+.*?(?=- \[ \])' tasks.md | \
            grep -vP '(FR|NFR|ADR)' && exit 1 || true
      - name: Check orphan FR
        run: |
          # Проверка: каждый FR имеет Task и Test
          for fr in $(grep -oP 'FR-\d+' specs/*/spec.md); do
            grep -q "$fr" tasks.md || exit 1
            grep -q "$fr" test-specs/*/tests.md || exit 1
          done
      - name: Check NFR-SEC → TM link
        run: |
          # Проверка: каждый NFR-SEC ссылается на TM-NNN
          grep -oP 'NFR-SEC-\d+' specs/*/spec.md | while read nfr; do
            grep -q "$nfr" threat-model.md || exit 1
          done
```

> Эта проверка превращает Definition of Done из декларации в **проверяемое условие**.
> Без CI — DoD проверяется вручную при code review.

---

## 🛡️ Conflict Handling: TM vs Observability

Ключевой механизм безопасности — обработка конфликтов между Threat Model и Observability Contract:

| Конфликт | Пример | Решение |
|----------|--------|---------|
| PII в телеметрии | TM выявил `user.email` в логе, OC не предусматривает redaction | Добавить NFR-OBS на redaction или убрать поле из логов |
| Telemetry как вектор | TM выявил утечку через Langfuse SaaS | Добавить TB-05 + шифрование + retention policy |
| Missing signal | OC требует метрику, TM не видит защиты | Добавить NFR-SEC на защиту метрики |
| SLO vs constraints | SLO требует OpenTelemetry, constraints запрещает новые зависимости | Запросить разрешение, предложить альтернативу |

> Этот механизм реализован в `system-prompts/M5-threat-model.md` и `M8-tasks-apply-and-traceability.md`.
> При обнаружении конфликта pipeline **останавливается** и предлагает пользователю выбор решения.
