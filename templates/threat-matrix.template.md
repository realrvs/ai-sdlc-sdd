# Threat Matrix Template

> Скопируйте этот шаблон в `threat-model.md` → раздел `## 5. Матрица угроз`.
> Заполните после выбора методологии (см. M5-threat-model.md).

---

## Матрица угроз с Risk Scoring

> **Risk = Likelihood (1–5) × Impact (1–5). Max = 25.**
> **Top-3** — угрозы с наивысшим Risk для immediate mitigation.

| ID | Категория | Угроза | Вектор | Актив | TB | L | I | Risk | Контроль | Requirement |
|----|-----------|--------|--------|-------|----|---|---|------|----------|-------------|
| TM-01 | [категория] | [описание угрозы] | [вектор атаки] | [актив] | [TB-NNN] | [1-5] | [1-5] | **[L×I]** | [контроль] | NFR-SEC-NNN |
| TM-02 | [категория] | [описание угрозы] | [вектор атаки] | [актив] | [TB-NNN] | [1-5] | [1-5] | **[L×I]** | [контроль] | NFR-SEC-NNN |
| TM-03 | [категория] | [описание угрозы] | [вектор атаки] | [актив] | [TB-NNN] | [1-5] | [1-5] | **[L×I]** | [контроль] | NFR-SEC-NNN |

### Top-3 для immediate mitigation
1. **TM-XX (Risk XX)** — [краткое описание] → NFR-SEC-XXX, P0
2. **TM-XX (Risk XX)** — [краткое описание] → NFR-SEC-XXX, P0
3. **TM-XX (Risk XX)** — [краткое описание] → NFR-SEC-XXX, P1

### Risk Scoring Reference
| Score | Level | Action |
|-------|-------|--------|
| 20–25 | Critical | Immediate mitigation, block release |
| 12–19 | High | Mitigation in current sprint |
| 6–11 | Medium | Mitigation in next sprint |
| 1–5 | Low | Backlog, monitor |
