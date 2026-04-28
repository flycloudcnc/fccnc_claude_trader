# DDD Template: Integration Contract

## Usage

Use this template to define the **contract between two bounded contexts**. Each integration point gets its own contract document. The template supports several integration styles — choose the section(s) that apply, delete the rest:

- **Synchronous request/response** (REST or gRPC)
- **Asynchronous events** (pub/sub or message queue)
- **CQRS with Materialized Views** (events on the write side, projected read models)
- **Shared Kernel** (only with an ADR)

The machine-readable contract — OpenAPI 3.1 for REST, AsyncAPI 2.6 for events — is the runtime source of truth and lives in `domains/<ctx>/api-contract.yaml` and/or `domains/<ctx>/events.asyncapi.yaml`. **This markdown describes the *why and how*; the YAML describes the bytes on the wire.** When they disagree, the YAML wins for runtime; fix the markdown.

Event payload schemas are defined in **`architecture/event-catalog.md`** (the single source of truth) and in `events.asyncapi.yaml`. This contract references events by name.

---

## Template

```markdown
# Integration Contract: [Context A] <-> [Context B]

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]
**Direction:** [A -> B / B -> A / Bidirectional]
**Style:** [Sync request/response | Async events | CQRS+materialized view | Shared kernel]

## Overview

[1-2 sentences: what integration this contract covers and why these contexts communicate.]

## Integration Pattern

| Attribute | Value |
|----------|-------|
| **Style** | [from above] |
| **Upstream** | [Which context provides data / emits events / serves the API] |
| **Downstream** | [Which context consumes data / calls the API] |
| **Coupling** | [Loose / Medium / Tight] |
| **Relationship type** | [Published Language / Open Host Service / ACL / Conformist / Shared Kernel] |
| **Machine-readable spec** | [path to api-contract.yaml or events.asyncapi.yaml] |

> If `Style = Shared Kernel`, link the ADR that authorized it (HR-2).

---

## Section A — Synchronous (REST / gRPC)

> Delete this section if the integration is async-only.

### Operations

| Operation ID | Method/Verb | Path or RPC | Idempotent? | Auth |
|--------------|-------------|-------------|-------------|------|
| [opId] | [GET/POST/...] | [/resource or rpc.Service/Method] | [Yes/No] | [scheme] |

### Per-Operation Detail

#### [operationId]

- **Maps to Command/Query in:** [bounded-context.md#commands]
- **Request schema:** see `api-contract.yaml#/components/schemas/[Model]`
- **Response schema:** see `api-contract.yaml#/components/schemas/[Model]`
- **Domain invariants enforced:** [IDs from aggregate.md]
- **Side effects (events emitted):** [EventName(s)]
- **Failure modes:**

| Status | Condition | Caller action |
|--------|-----------|---------------|
| 4xx    | [Validation / invariant violation] | [What to do] |
| 5xx    | [Server error] | [Retry policy] |

### SLA

| Attribute | Value |
|-----------|-------|
| Target latency (p99) | [ms] |
| Target throughput | [rps] |
| Availability | [%] |
| Backpressure | [How the upstream signals overload] |

---

## Section B — Asynchronous Events

> Delete this section if the integration is sync-only.

### Events Exchanged

| Event | Direction | Channel | Delivery | Ordering |
|-------|-----------|---------|----------|----------|
| [EventName] | A -> B | [context.aggregate.EventName] | [at-least-once / at-most-once / exactly-once] | [Guaranteed / Best-effort] |

### Per-Event Notes

#### [EventName]

- **Payload:** defined in `architecture/event-catalog.md` and `events.asyncapi.yaml`.
- **Trigger (publisher side):** [Which command emits it]
- **Subscriber reaction:** [What the downstream does]
- **Idempotency key:** [Field used to deduplicate retries]
- **Versioning:** [How payload schema changes are handled — additive only on existing channels (HR-7)]

### Error & Retry

| Scenario | Publisher behaviour | Subscriber behaviour |
|----------|--------------------|----------------------|
| Delivery failure | [Retry / DLQ / fire-and-forget] | [Wait / Compensate] |
| Invalid payload | [Reject / Log / DLQ] | [Skip / Log / Alert] |
| Duplicate delivery | [N/A] | [Use idempotency key] |

---

## Section C — CQRS Materialized Views

> Delete this section if the downstream does not project events into a read model.

### Views

| View | Source events | Purpose | Refresh | Staleness tolerance | Storage |
|------|--------------|---------|---------|---------------------|---------|
| [ViewName] | [EventName, ...] | [Question it answers] | [Sync / async / periodic] | [real-time / eventually consistent] | [DB table / cache / file] |

### Projection Rules

| View | Event | Projection rule (upsert / append / recalculate) |
|------|-------|------------------------------------------------|
| [View] | [Event] | [Rule] |

### Rebuild Strategy

[How to rebuild every view from scratch — typically: replay all events from the event store / re-query the upstream API. Required for every view.]

### View Schema Excerpt

```json
{
  "view": "[view_name]",
  "data": { "...": "..." },
  "last_projected_event_id": "evt_...",
  "projected_at": "YYYY-MM-DDTHH:MM:SSZ"
}
```

---

## Anti-Corruption Layer (if applicable)

| Upstream type/field | Downstream type/field | Translation |
|---------------------|----------------------|-------------|
| [Upstream] | [Downstream] | [How the ACL translates] |

> An ACL is required when the downstream chooses **not** to conform to the upstream model (`Relationship type` is **not** `Conformist`).

---

## Cross-cutting

### Idempotency

| Direction | Mechanism |
|-----------|-----------|
| A -> B | [Idempotency key / dedupe table / event id check] |

### Versioning Policy

- **Additive changes** (new optional fields, new operations, new events): minor bump, no consumer break.
- **Breaking changes**: new operation/event name + deprecation window; ADR required.

### Security

| Concern | Approach |
|---------|----------|
| AuthN | [Scheme — bearer JWT / mTLS / API key] |
| AuthZ | [Which scopes / roles] |
| PII | [Fields that must be redacted in logs] |

### Observability

| Signal | Where |
|--------|-------|
| Trace propagation | [Headers / context fields] |
| Metrics | [What both sides emit] |
| Logs | [Correlation id field] |

---

## Code References

| Component | Context | File Path |
|-----------|---------|-----------|
| [Server / event publisher] | [Upstream] | [path] |
| [Client / event subscriber / projector] | [Downstream] | [path] |
| [Read-model store / view] | [Downstream] | [path] |
| [ACL / translator] | [Downstream] | [path] |

## Related Docs

| Doc | Path |
|-----|------|
| Upstream Context | [domains/<upstream>/bounded-context.md] |
| Downstream Context | [domains/<downstream>/bounded-context.md] |
| OpenAPI spec | [domains/<upstream>/api-contract.yaml] |
| AsyncAPI spec | [domains/<upstream>/events.asyncapi.yaml] |
| Event Catalog | [architecture/event-catalog.md] |
| Authorizing ADR (if shared kernel or breaking change) | [architecture/adr/NNNN-...md] |
```

---

## Guidelines

1. **One contract per integration point** — if two contexts have multiple integration points, create separate contracts.
2. **Pick a style explicitly** — `Style:` row drives which sections you keep. Delete unused sections; do not leave them as `N/A`.
3. **YAML is the runtime source of truth** — markdown describes intent; OpenAPI/AsyncAPI describes the wire.
4. **Don't redefine event payloads here** — reference `event-catalog.md` and the AsyncAPI file by event name.
5. **Versioning policy is mandatory** — every contract must state how breaking changes are handled (HR-7).
6. **Shared Kernel requires an ADR** — link it from the Integration Pattern table; otherwise reject in review (HR-2).
7. **Materialized views are optional** — only document Section C when the downstream actually maintains a read model.
8. **Name files by the two contexts** — e.g., `trading-risk.md`, `equipment-event.md`. Lives at `architecture/contracts/<a>-<b>.md`.
9. **Layer 3 doc** — load only when implementing or debugging this specific integration point.
