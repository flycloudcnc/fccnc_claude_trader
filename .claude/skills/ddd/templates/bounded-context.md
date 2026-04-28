# DDD Template: Bounded Context Specification

## Usage

Use this template to define a single bounded context. Each bounded context represents an autonomous domain area with its own models, rules, and language. Keep specs **concise and token-efficient** for AI consumption.

---

## Template

```markdown
# Bounded Context: [Context Name]

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]
**Owner:** [Team or person responsible]

## Purpose

[1-2 sentences describing what this context is responsible for and why it exists.]

## Ubiquitous Language

| Term | Definition |
|------|-----------|
| [Term] | [Precise definition within this context] |

## Aggregates

> **Note:** Entities and Value Objects live *inside* aggregates — they are listed here under their owning aggregate, not as standalone context-level items. See "DDD Hierarchy Reference" at the bottom.

| Aggregate Root | Child Entities | Value Objects | Invariants |
|---------------|----------------|---------------|-----------|
| [Root Entity (ID type)] | [Child entities with own identity] | [Identity-less objects defined by attributes] | [Business rules that must always hold] |

## Domain Events

| Event | Trigger | Payload | Consumers |
|-------|---------|---------|-----------|
| [EventName] | [What causes it] | [Key data fields] | [Which contexts consume it] |

## Commands

| Command | Handler | Preconditions | Side Effects |
|---------|---------|--------------|-------------|
| [CommandName] | [Which aggregate/service] | [What must be true] | [Events emitted, state changes] |

## Domain Services

| Service | Responsibility | Dependencies |
|---------|---------------|-------------|
| [Name] | [What logic it encapsulates] | [Other services/repos it uses] |

## Repositories

| Repository | Aggregate | Key Operations |
|-----------|-----------|---------------|
| [Name] | [Which aggregate it persists] | [find_by_x, save, delete, etc.] |

## External Dependencies

| Dependency | Type | Purpose |
|-----------|------|---------|
| [Context/System] | [Event / API / Shared Kernel] | [Why this context depends on it] |

## Invariants / Business Rules

1. [Rule description — must always be true within this context]
2. [Another rule]

## Code References

| Component | File Path |
|-----------|----------|
| [Entity/Service/Repo] | [path/to/file.py] |

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md] |
| [Aggregate 1] Spec | [aggregates/<this-context>/<aggregate-1>.md] |
| [Aggregate 2] Spec | [aggregates/<this-context>/<aggregate-2>.md] |
| Contract: [Other Context] | [contracts/<this-context>-<other-context>.md] |
| Event Catalog | [event-catalog.md] |
```

---

## DDD Hierarchy Reference

This shows how DDD building blocks relate within a bounded context:

```
Domain
 └── Bounded Context (1..N per domain)
      ├── Ubiquitous Language (scoped to this context)
      ├── Aggregate (1..N)
      │    ├── Aggregate Root (1) — this IS an Entity
      │    ├── Entity (0..N)      — child entities with own identity
      │    └── Value Object (0..N) — identity-less, defined by attributes
      │    └── emits → Domain Events
      ├── Domain Service (0..N)   — cross-aggregate logic
      ├── Domain Events
      └── Integration Contract    — connects to other contexts
```

**Key rule:** Entities and Value Objects always live *inside* an Aggregate — they never float freely at the context level. All access to child entities goes through the Aggregate Root.

**Domain Services** contain logic that spans multiple aggregates within the same context and doesn't naturally belong to either aggregate. They operate *through* aggregate roots, never bypassing them.

---

## Guidelines

1. **One context per file** — never combine multiple bounded contexts.
2. **Keep it under 200 lines** — if longer, the context may need to be split.
3. **Use tables over prose** — tables are more token-efficient and scannable.
4. **Link to code** — always include Code References so the AI can locate implementations.
5. **Version it** — update the version when the context's model changes.
6. **Terms stay local** — if the same word means different things in different contexts, define it separately in each.
7. **Entities belong to aggregates** — never define standalone entities at the context level. Every entity must be part of an aggregate.
8. **Domain Services for cross-aggregate logic** — if logic doesn't fit in a single aggregate, use a Domain Service that coordinates through aggregate roots.
9. **Layer 2 doc** — summarize aggregates here; full detail lives in Layer 3 aggregate docs. Cross-reference via Related Docs.
