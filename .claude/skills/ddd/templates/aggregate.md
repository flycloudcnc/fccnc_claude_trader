# DDD Template: Aggregate Definition

## Usage

Use this template for a **deep-dive into a single aggregate**. This is the most granular DDD spec — it defines the consistency boundary, state machine, and all operations for one aggregate root. Use when an AI agent needs to implement or modify behavior within a specific aggregate.

---

## Template

```markdown
# Aggregate: [Aggregate Root Name]

**Bounded Context:** [Which context this belongs to]
**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

## Purpose

[1-2 sentences: what real-world concept this aggregate models and why it exists as a consistency boundary.]

## Aggregate Root

> The Aggregate Root **is** an Entity — it has a unique identity and is the only entry point for external code. Outside code must never modify child entities or value objects directly; all changes go through the root.

| Property | Type | Description | Constraints |
|----------|------|-------------|-------------|
| [id] | [UUID/int/etc.] | [Identity field] | [Required, unique] |
| [field] | [type] | [What it represents] | [Nullable? Range? Format?] |

## Child Entities

> Entities have their own **identity** (an ID that persists over time). Even if all attributes change, the entity remains the "same" entity. They live inside the aggregate and are only accessible through the root.

| Entity | Identity | Relationship | Description |
|--------|----------|-------------|-------------|
| [Name] | [ID type] | [1:N / 1:1] | [What it represents within this aggregate] |

## Value Objects

> Value Objects have **no identity** — they are defined entirely by their attributes. Two value objects with the same attributes are interchangeable. They are immutable.

| Value Object | Used By | Attributes | Validation |
|-------------|---------|-----------|------------|
| [Name] | [Which entity uses it] | [Fields] | [Rules] |

## State Machine

| State | Description | Transitions To | Trigger |
|-------|------------|----------------|---------|
| [State] | [What this state means] | [Next valid states] | [Command/Event that causes transition] |

## Commands (Write Operations)

| Command | Preconditions | State Changes | Events Emitted |
|---------|--------------|--------------|----------------|
| [CommandName] | [What must be true before execution] | [What changes in the aggregate] | [Events published after success] |

## Queries (Read Operations)

| Query | Returns | Description |
|-------|---------|-------------|
| [QueryName] | [Return type] | [What data it provides] |

## Invariants

These rules must **always** hold true within this aggregate boundary:

1. [Invariant description]
2. [Another invariant]

## Domain Events Emitted

| Event | When | Payload |
|-------|------|---------|
| [EventName] | [After which command/state change] | [Key fields in the event] |

## Domain Events Consumed

| Event | Source Context | Reaction |
|-------|---------------|----------|
| [EventName] | [Publishing context] | [What this aggregate does in response] |

## Concurrency / Consistency

- **Consistency boundary:** [What is guaranteed to be consistent within this aggregate]
- **Concurrency strategy:** [Optimistic locking / versioning / last-write-wins]
- **Transaction scope:** [What is included in a single transaction]

## Code References

| Component | File Path |
|-----------|----------|
| [Aggregate Root] | [path/to/aggregate.py] |
| [Repository] | [path/to/repository.py] |
| [Domain Service] | [path/to/service.py] |

## Related Docs

| Doc | Path |
|-----|------|
| Parent Context | [contexts/<parent-context>.md] |
| Event Catalog | [event-catalog.md] — see [EventName1], [EventName2] |
| Contract: [Other Context] | [contracts/<context>-<other-context>.md] |
```

---

## Guidelines

1. **One aggregate per file** — never combine multiple aggregates.
2. **State machines are mandatory** — if the aggregate has states, document all valid transitions.
3. **Invariants are the most important section** — these are the rules the AI must never violate.
4. **Keep under 150 lines** — if larger, the aggregate may be doing too much.
5. **Commands produce events** — every write operation should emit at least one domain event.
6. **Consistency boundary is sacred** — document exactly what is guaranteed to be consistent.
7. **Root is the gatekeeper** — all modifications to child entities and value objects must go through the aggregate root. External code never reaches in to change internals directly.
8. **Entity vs Value Object** — if it needs an ID to be tracked over time, it's an Entity. If it's defined purely by its attributes and is interchangeable, it's a Value Object.
9. **Layer 3 doc** — this is the deepest detail level. Load only when implementing or modifying this specific aggregate. Always link back to the parent context doc.
