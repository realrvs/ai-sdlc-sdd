# M7: API Contract & Test Specs

## Формат api-contract.yaml

```yaml
openapi: "3.1.0"
info:
  title: <API Name>
  version: "1.0.0"
  description: <1-2 строки>
security:
  - <scheme>: []
components:
  securitySchemes:
    <scheme>:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    <SchemaName>:
      type: object
      required: [...]
      properties:
        ...
    ErrorResponse:
      type: object
      required: [error_code, message, trace_id]
      properties:
        error_code:
          type: string
        message:
          type: string
        trace_id:
          type: string
          description: W3C trace-id для корреляции с трейсом
paths:
  /<endpoint>:
    <method>:
      summary: <что делает>
      operationId: <snake_case_unique>
      security:
        - <scheme>: []
      parameters:
        - name: traceparent
          in: header
          required: false
          schema:
            type: string
          description: W3C Trace Context (обязателен для сквозной диагностики)
        - name: X-Request-ID
          in: header
          required: false
          schema:
            type: string
      requestBody:
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/<RequestSchema>"
            examples:
              default:
                value: { ... }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/<ResponseSchema>"
        "400":
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "429":
          description: Too Many Requests
        "500":
          description: Internal Server Error
```

### Правила API Contract
| Требование | Детали |
|-----------|--------|
| OpenAPI | 3.1.0 YAML |
| Ошибки | **Graduated approach:** базовые CRUD: 400, 404, 500; расширенные: +401, 403, 409, 422, 429 |
| Security | `securitySchemes` + `security` на каждый защищённый endpoint |
| Examples | `examples.value` на каждый endpoint (request + response) |
| Schemas | Все request/response body — именованные `$ref` schemas |
| **Correlation** | `traceparent` и `X-Request-ID` в параметрах всех endpoint'ов |
| **Error trace_id** | `ErrorResponse.trace_id` обязателен — для support-диагностики |

---

## Формат test-specs/{domain-name}/tests.md

```markdown
## Test Suite: <domain-name>
### TC-NNN: <test-name>
- **Type**: Functional | Integration | E2E | Performance | Security | **Observability**
- **Covers**: FR-NNN / NFR-NNN / NFR-SEC-NNN / **NFR-OBS-NNN**
- **TM Ref**: TM-NNN   # если тест покрывает угрозу
- **Priority**: P0 (critical) | P1 (high) | P2 (medium) | P3 (low)
#### Steps
1. Given <setup>
2. When <action>
3. Then <expected result>
#### Data
- Input: { ... }
- Expected Output: { ... }
#### Observability Assertions
> Заполняется для тестов типа Observability и любых тестов, проверяющих runtime-путь.
- **Trace:** span `POST /orders` существует, имеет `order.id`, `trace_id`
- **Metrics:** `http.server.request.duration{status_code="200"}` увеличился
- **Logs:** в Loki есть запись с `trace_id` = <ожидаемый>
- **Security:** в логах/спанах нет PII (`pii_redacted=true`)
```

### Правила тестов
- Каждый FR/NFR/NFR-SEC/**NFR-OBS** должен иметь **≥1 тест**
- ID: `TC-NNN` (3 цифры)
- Тип теста определяется типом требования:
  - FR → Functional / Integration / E2E
  - NFR → Performance / Security
  - NFR-SEC → **Security** (обязательно, с явной ссылкой на TM-NNN)
  - **NFR-OBS → Observability** — assertions на спанах/метриках/логах
- **Security-тесты**: любой тест, покрывающий TM-NNN, обязан включать
  Observability Assertion об **отсутствии PII** в телеметрии
- Priority: P0 для critical path и угроз с уровнем **Critical**; P1 для остальных
