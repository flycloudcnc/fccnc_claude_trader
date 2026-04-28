# DDD Template: Machine-Readable API Contract (OpenAPI / AsyncAPI)

## Usage

Use this template to produce the **machine-readable** contract for a bounded context — what other contexts can call (synchronous) or subscribe to (asynchronous). This complements the markdown `integration-contract.md` (which describes *intent and patterns*) by giving CI a file it can lint, diff, and use for contract tests.

- Synchronous (REST/gRPC) -> **OpenAPI 3.1 YAML**
- Asynchronous (events / messages) -> **AsyncAPI 2.6 YAML**

One file per published interface. Lives at `domains/<context>/api-contract.yaml` for a single REST API, or `domains/<context>/events.asyncapi.yaml` for the published event surface. A context that both serves a REST API and publishes events keeps both files.

---

## OpenAPI 3.1 Template (REST)

```yaml
openapi: 3.1.0
info:
  title: "[Context Name] API"
  version: "X.Y.Z"
  description: |
    Published interface for the [Context] bounded context.
    Source of truth for synchronous integration. Markdown counterpart:
    integration-contract.md.
  contact:
    name: "[Owning team]"
servers:
  - url: "https://[host]/[context]/v1"
    description: "[env]"

tags:
  - name: "[Aggregate or capability group]"
    description: "[1 line]"

paths:
  /[resource]:
    get:
      tags: ["[Aggregate]"]
      operationId: "[verbResource]"
      summary: "[1 line]"
      parameters:
        - in: query
          name: "[name]"
          required: false
          schema: { type: string }
      responses:
        "200":
          description: "OK"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/[ResponseModel]"
        "4XX":
          $ref: "#/components/responses/ClientError"
        "5XX":
          $ref: "#/components/responses/ServerError"
    post:
      tags: ["[Aggregate]"]
      operationId: "[verbResource]"
      summary: "[1 line — must map to a Command in bounded-context.md]"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/[CommandModel]"
      responses:
        "202":
          description: "Accepted — command queued; emits [EventName]"
        "409":
          description: "Invariant violation"

components:
  schemas:
    "[ResponseModel]":
      type: object
      required: [id]
      properties:
        id: { type: string, format: uuid }
        # Mirror fields from the aggregate's read model, not internals.

    "[CommandModel]":
      type: object
      required: [...]
      properties:
        # Mirror the Command preconditions in bounded-context.md.

  responses:
    ClientError:
      description: "Domain rule or input validation failure"
      content:
        application/json:
          schema:
            type: object
            required: [code, message]
            properties:
              code: { type: string }
              message: { type: string }
              invariant: { type: string, description: "Name of violated invariant, if any" }
    ServerError:
      description: "Unexpected server error"

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

---

## AsyncAPI 2.6 Template (Events)

```yaml
asyncapi: 2.6.0
info:
  title: "[Context Name] Event Stream"
  version: "X.Y.Z"
  description: |
    Published events from the [Context] bounded context.
    Payload schemas are the single source of truth — the markdown
    domain-event-catalog.md references this file by event name.

defaultContentType: application/json

servers:
  bus:
    url: "[broker-host]"
    protocol: "[kafka / rabbitmq / nats / sqs]"
    description: "[env]"

channels:
  "[context].[aggregate].[EventName]":
    description: "[1-line trigger description]"
    publish:
      operationId: "publish[EventName]"
      message:
        $ref: "#/components/messages/[EventName]"

components:
  messages:
    "[EventName]":
      name: "[EventName]"
      title: "[Past-tense human label]"
      summary: "[Why this event exists]"
      contentType: application/json
      headers:
        $ref: "#/components/schemas/EventHeaders"
      payload:
        $ref: "#/components/schemas/[EventName]Payload"

  schemas:
    EventHeaders:
      type: object
      required: [event_id, event_type, occurred_at, version]
      properties:
        event_id: { type: string, format: uuid }
        event_type: { type: string }
        occurred_at: { type: string, format: date-time }
        version: { type: string, description: "Semantic version of the payload schema" }
        correlation_id: { type: string }
        causation_id: { type: string }

    "[EventName]Payload":
      type: object
      required: [...]
      properties:
        # Field set MUST match the row in domain-event-catalog.md.
```

---

## Guidelines

1. **Spec is the source of truth** — when the markdown contract and the YAML disagree, the YAML wins for runtime; fix the markdown.
2. **One operation per command, one channel per event** — never bundle multiple commands behind one endpoint or multiple events on one channel.
3. **Path / channel naming**:
   - REST: `/<resource>` plural nouns; verbs are HTTP methods, not URL segments.
   - Events: `<context>.<aggregate>.<EventName>` — past tense, dotted.
4. **Schemas must mirror the domain model** — every `CommandModel` corresponds to a Command row in `bounded-context.md`; every event payload corresponds to a row in `domain-event-catalog.md`.
5. **Versioning**: bump `info.version` (semver). Breaking changes require a new ADR and an update to `integration-contract.md`.
6. **Lint in CI** — run `spectral lint` (or equivalent) and fail the build on rule violations.
7. **Contract tests** — consumer side validates against the same YAML; never against a hand-written stub.
8. **Pair with markdown** — `integration-contract.md` describes *why* and *how* contexts interact; the YAML describes *exactly what bytes go on the wire*. Both are required.
9. **No internal types** — never expose ORM entities or write-model state directly; expose read-model / command DTOs only.
10. **Security is mandatory** — every OpenAPI doc declares a `securitySchemes` and a default `security` block.
