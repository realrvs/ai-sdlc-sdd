# M2: Proposal

## Формат proposal.md

```markdown
## Why
[Проблема + бизнес-контекст + метрики]

## What Changes
[Список изменений — bullet points]

## Impact
- Затронутые capabilities: [...]
- Затронутые команды: [...]
- Целевые метрики: [...]
- Security-профиль: [...]   # краткая выжимка из security_context
- **Operability-профиль:** [...]  # выжимка из observability_context: SLO, стек, что уже инструментировано

## Operability Promise
> Как мы узнаем, что фича работает в production?

- **SLO:** [latency p95, availability, error rate]
- **Ключевые сигналы:** [RED-метрики, бизнес-метрики, критичные логи]
- **Что НЕ покрыто наблюдаемостью:** [явно перечислить, чтобы не было иллюзий]
```

### Правила
- **Why** — 2-3 абзаца с бизнес-метриками (до/после)
- **Operability Promise** — обязателен, если `observability_context.not_applicable != true`
- **Impact** — Security-профиль обязателен, если `security_context` не `baseline`
