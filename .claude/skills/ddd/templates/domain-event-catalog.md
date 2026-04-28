# DDD Template: Domain Event Catalog

## Usage

Use this template to maintain a **system-wide catalog of all domain events**. This is the central reference for event-driven communication between bounded contexts. Event payload schemas are defined **here** as the single source of truth — integration contracts reference events from this catalog rather than redefining them.

For how events are used between specific contexts (command/query paths, materialized views, projections), see the **Integration Contract** template.

---

## Template

```markdown
# Domain Event Catalog

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

## Event Summary

| Event | Publisher Context | Subscribers | Materialized Views | Criticality |
|-------|-----------------|-------------|-------------------|-------------|
| [EventName] | [Context] | [Context1, Context2] | [View1, View2] | [High/Medium/Low] |

## Event Definitions

### [EventName]

- **Publisher:** [Bounded context that emits this event]
- **Trigger:** [What causes this event to be emitted]
- **Criticality:** [High / Medium / Low — what happens if this event is lost?]

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| [field_name] | [type] | [Yes/No] | [What this field contains] |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| [Subscribing context] | [Command-side reaction, if any] | [ViewName or N/A] | [Retry / Skip / Dead Letter] |

**Integration Contracts:** [List of contract files that reference this event, e.g., `trading-risk-contract.md`]

---

### [Another EventName]

[Repeat the same structure for each event]

---

## Event Flow Diagrams

```text
[Text-based sequence showing event chains and view projections]

Trade Executed
  -> Trading: emits TradeExecuted
    -> Risk: projects into DrawdownView, updates position
    -> Account: projects into PortfolioSummaryView
    -> Notification: sends trade confirmation (no view)
```

## Event Conventions

| Convention | Rule |
|-----------|------|
| **Naming** | [Past tense verb, e.g., TradeExecuted, SignalGenerated] |
| **Payload format** | [JSON / dict / dataclass] |
| **Delivery guarantee** | [At-least-once / at-most-once / exactly-once] |
| **Ordering** | [Whether event ordering is guaranteed] |
| **Idempotency** | [Whether subscribers must handle duplicate events] |
| **Schema ownership** | [Payloads are defined in this catalog — integration contracts reference, not redefine] |

## Dead Letter / Error Events

| Error Event | Source | Condition | Handler |
|------------|--------|-----------|---------|
| [ErrorEventName] | [Context] | [When this error event is emitted] | [What handles it] |

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md] |
| [Context A] Spec | [contexts/<context-a>.md] |
| Contract: [A] <-> [B] | [contracts/<context-a>-<context-b>.md] |
```

---

## Guidelines

1. **One catalog per system** — all events across all contexts in one file.
2. **Past tense naming** — events describe something that already happened (TradeExecuted, not ExecuteTrade).
3. **Payload is defined here, referenced elsewhere** — this catalog owns event schemas. Integration contracts reference events by name and link back here. Do not duplicate payload definitions.
4. **Every event needs at least one subscriber** — orphan events are dead code.
5. **Document which views each event feeds** — the summary table and subscriber sections must note materialized views that project from each event.
6. **Document failure handling** — what happens when a subscriber or projection fails to process an event.
7. **Keep the summary table at the top** — AI agents scan this first to find relevant events and their downstream views.
8. **Cross-reference integration contracts** — each event definition should list which integration contract files use it.
9. **Layer 1-2 doc** — the summary table is Layer 1 (scan to orient). Individual event details are Layer 2 (load only when needed). If the catalog exceeds 200 lines, split into per-context event files.
