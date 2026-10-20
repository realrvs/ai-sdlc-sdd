# M4: Design

## Формат design.md

```markdown
## Context
[1-2 абзаца: контекст и ограничения]

## Goals / Non-Goals
**Goals:**
- [...]
**Non-Goals:**
- [...]

## Decisions
### Компонентная архитектура
[Описание слоёв/компонентов]

### Потоки данных
[Sequence/flow описание или Mermaid]

### Технологический стек
| Слой | Технология | Обоснование |
|---|---|---|

### Точки инструментирования (NEW)
> Заполняется на основе Observability Contract из spec.md.

| Сигнал | Где создаётся (файл/класс) | Тип | Auto/Manual | Атрибуты |
|--------|---------------------------|-----|-------------|----------|
| `http.server.request.duration` | OTel auto-instrumentation (FastAPI middleware) | metric | auto | см. spec § Observability |
| `order.validated` | `OrderService.validate()` | span | manual | `order.id`, `validation.result` |

### Проброс корреляции (NEW)
- **W3C `traceparent`:** [как передаётся между компонентами]
- **`X-Request-ID`:** [если используется]
- **Service A → Service B:** [как `traceparent` попадает из вызова в вызов]

## Risks / Trade-offs
- **Риск**: … → **Митигация**: …
- **Trade-off**: … → **Митигация**: …

## Migration Plan
1. Шаг 1
2. Шаг 2
…

## Open Questions
- [ ] Вопрос 1
- [ ] Вопрос 2
```

### Диаграммы
- Использовать Mermaid (`mermaid` код-блок с явным типом `mermaid`)
- Обязательные диаграммы (если применимо):
  - **Component diagram** — архитектура
  - **Sequence diagram** — ключевой поток
  - **Data flow diagram** — если `security_context` не `baseline` (вход для threat-model)
  - **Observability flow diagram** — если `observability_context` не `not_applicable`:
    показать, как сигналы от источников идут в бэкенды (OTel Collector → Prometheus/Loki → Grafana)
