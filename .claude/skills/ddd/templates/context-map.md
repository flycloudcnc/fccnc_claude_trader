# DDD Template: Context Map

## Usage

Use this template to define the **system-level view** of all bounded contexts and their relationships. This is the master navigation document that AI agents use to understand which context owns what and how contexts integrate.

---

## Template

```markdown
# Context Map

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

## Bounded Contexts Overview

| Context | Purpose | Upstream Of | Downstream Of |
|---------|---------|------------|---------------|
| [Name] | [1-sentence purpose] | [Contexts it feeds] | [Contexts it depends on] |

## Relationship Types

Legend for relationship patterns used in this system:

| Pattern | Description |
|---------|------------|
| **Published Language (PL)** | Context exposes a well-defined API or event schema |
| **Open Host Service (OHS)** | Context provides a service for multiple consumers |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream models to protect its own model |
| **Shared Kernel (SK)** | Two contexts share a small common model |
| **Conformist (CF)** | Downstream adopts upstream's model as-is |
| **Customer-Supplier (CS)** | Upstream prioritizes downstream's needs |

## Context Relationships

| Upstream | Downstream | Pattern | Integration Mechanism | Description |
|----------|-----------|---------|----------------------|-------------|
| [Context A] | [Context B] | [PL/OHS/ACL/SK/CF/CS] | [Events / API / Shared DB / Direct Call] | [What flows between them] |

## Data Flow Diagram

```text
[Text-based diagram showing directional data flow between contexts]

  Context A ---(EventX)---> Context B
       |                        |
       |                        v
       +----(API call)---> Context C
```

## Dependency Rules

1. [Rule about allowed dependencies, e.g., "Risk must never depend on Notification"]
2. [Rule about data ownership, e.g., "Only Trading context can modify Trade entities"]
3. [Rule about event direction, e.g., "Market Data publishes events, never consumes them"]

## Shared Models (if any)

| Model | Owned By | Shared With | Sharing Mechanism |
|-------|---------|-------------|-------------------|
| [Model name] | [Owning context] | [Consuming contexts] | [Shared Kernel / Published Language] |

## Anti-Corruption Layers

| ACL Location | Upstream Context | Translation |
|-------------|-----------------|-------------|
| [Where the ACL lives] | [What it protects against] | [How it translates upstream models] |

## Related Docs

| Doc | Path |
|-----|------|
| Ubiquitous Language | [ubiquitous-language.md] |
| [Context A] Spec | [contexts/<context-a>.md] |
| [Context B] Spec | [contexts/<context-b>.md] |
| Event Catalog | [event-catalog.md] |
```

---

## Guidelines

1. **One context map per system** — this is the single source of truth for context relationships.
2. **Keep it visual** — use text diagrams to show data flow direction.
3. **Direction matters** — always specify upstream vs. downstream clearly.
4. **Dependency rules are constraints** — AI agents must not violate these when implementing features.
5. **Update on every context change** — when a new context is added or relationships change, update this map first.
6. **This is the AI's navigation map** — an AI agent should be able to read this and know which context spec to load for any given task.
7. **Layer 1 doc** — always load this first. It links to all Layer 2 bounded context docs via Related Docs.
