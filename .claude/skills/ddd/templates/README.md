# DDD Documentation Templates

## Overview

These templates produce the artifacts of the **Governed AI Software Delivery** model (PDF: `doc/governed_ai_software_delivery.pdf`) — six layers from business intent to verification — using a **by-domain** repository layout. Templates are AI-friendly: concise, table-driven, and layered so an agent only loads what it needs.

## Templates by Layer

| Layer | Concern | Template | Output file (by-domain layout) | Max lines |
|-------|---------|----------|--------------------------------|-----------|
| 1. Business Intent | PRD / epics | (input only — not generated) | `doc/prd.md` | — |
| 2. Intent Translation | Use cases, term mapping, open questions | `intent-translation.md` | `domains/<ctx>/use-cases.md` | ~200 |
| 3. Domain Spec | Bounded contexts, aggregates, invariants, glossary, context map | `context-map.md`, `bounded-context.md`, `aggregate.md`, `ubiquitous-language.md`, `domain-event-catalog.md`, `architecture.md` | `architecture/*.md`, `domains/<ctx>/*.md` | per-template |
| 4. Contract | API + event schemas | `integration-contract.md`, `api-contract.md` | `architecture/contracts/<a>-<b>.md`, `domains/<ctx>/api-contract.yaml`, `domains/<ctx>/events.asyncapi.yaml` | per-template |
| 5. Delivery Execution | Implementation | (handled by `/implement`) | `services/`, `apps/`, `packages/`, `tests/` | — |
| 6. Verification & Governance | ADRs + AI rules | `adr.md`, `ai-development-rules.md`, `ai-review-checklist.md`, `ai-workflow-templates.md` | `architecture/adr/`, `ai-rules/*.md` | per-template |

## By-Domain Output Layout

The `/ddd` skill writes here. The layout matches the PDF's prescribed structure so that domain docs sit next to the code modules they describe.

```
project-root/
├── architecture/
│   ├── context-map.md                       <- Layer 3 (system overview)
│   ├── architecture.md                      <- Layer 3 (cross-cutting)
│   ├── ubiquitous-language.md               <- Layer 3 (system glossary)
│   ├── event-catalog.md                     <- Layer 3-4 (all events)
│   ├── adr/
│   │   └── NNNN-<slug>.md                   <- Layer 6
│   └── contracts/
│       └── <ctx-a>-<ctx-b>.md               <- Layer 4 (markdown contract)
│
├── domains/
│   ├── <context-a>/
│   │   ├── use-cases.md                     <- Layer 2
│   │   ├── ubiquitous-language.md           <- Layer 3 (context-local terms)
│   │   ├── bounded-context.md               <- Layer 3
│   │   ├── invariants.md                    <- Layer 3 (cross-aggregate rules)
│   │   ├── aggregates/
│   │   │   └── <aggregate>.md               <- Layer 3
│   │   ├── api-contract.yaml                <- Layer 4 (OpenAPI 3.1)
│   │   └── events.asyncapi.yaml             <- Layer 4 (AsyncAPI 2.6)
│   └── <context-b>/...
│
├── ai-rules/
│   ├── development-rules.md                 <- Layer 6
│   ├── review-checklist.md                  <- Layer 6
│   └── workflow-templates.md                <- Layer 6
│
├── services/ | apps/ | packages/            <- Layer 5 code
└── tests/                                   <- Layer 5 tests
```

The system-wide `architecture/ubiquitous-language.md` holds canonical terms; each domain's `domains/<ctx>/ubiquitous-language.md` only lists context-local terms or local meanings that override the system-wide one.

## Document Layers (loading strategy)

An AI agent working on a task should load **at most 3-4 files**, in this order:

```
Step 1 (Layer 1 docs)  -> "What exists?"          architecture/context-map.md + architecture/architecture.md + architecture/ubiquitous-language.md
Step 2 (Layer 2 doc)   -> "What's in this area?"  domains/<ctx>/bounded-context.md (+ event catalog summary if needed)
Step 3 (Layer 3 doc)   -> "How do I implement?"   one aggregate doc OR one integration contract
```

Rules:
- Always orient from the architecture-level docs first.
- Never load all aggregate files — only the one relevant to the task.
- If a task spans two contexts, load both `bounded-context.md` files but only the aggregates each side touches.
- The event catalog is split: scan the summary (Layer 1), read individual event detail only when needed.

## Cross-Reference Strategy

Every doc includes a `Related Docs` section with relative-path links. This creates a navigable graph — the AI traverses from any entry point to the exact detail it needs.

| Doc Type | Links To |
|----------|----------|
| Context Map | All bounded-context docs |
| Bounded Context | Its aggregate docs, its `use-cases.md`, contracts it participates in |
| Aggregate | Parent context doc + events emitted/consumed |
| Event Catalog | Contracts that reference each event + AsyncAPI file |
| Integration Contract | Both context docs + OpenAPI/AsyncAPI YAML + event catalog |
| Use Cases | Source PRD, context map, bounded-context, glossary, ADRs |
| ADR | Triggering use case + affected docs + superseded ADR |
| Ubiquitous Language | Contexts with overriding meanings |

**Anti-pattern: duplication.** Define once, reference by name everywhere:
- Event payloads -> `architecture/event-catalog.md` + `events.asyncapi.yaml`.
- Entity detail -> `aggregates/<aggregate>.md`; bounded-context only summarizes.
- Terms -> `architecture/ubiquitous-language.md`; per-domain only overrides.
- Integration wire format -> `api-contract.yaml` / `events.asyncapi.yaml`; markdown contract describes intent.

## DDD Hierarchy Reference

```
Domain
 └── Bounded Context (1..N per domain)
      ├── Ubiquitous Language (scoped to this context)
      ├── Aggregate (1..N)
      │    ├── Aggregate Root (1) — this IS an Entity
      │    ├── Entity (0..N)      — child entities with own identity
      │    └── Value Object (0..N) — identity-less, defined by attributes
      │    └── emits -> Domain Events
      ├── Domain Service (0..N)   — cross-aggregate logic
      ├── Domain Events
      └── Integration Contract    — connects to other contexts
```

Key rules:
1. Entities and Value Objects always live *inside* an Aggregate — never floating at the context level.
2. All access to child entities goes through the Aggregate Root.
3. Domain Services coordinate across aggregates but operate *through* aggregate roots.

## Generation Workflow

Generate top-down by layer. Each step is a separate Claude Code conversation to avoid context overflow.

```
Conv 1   Layer 3   architecture/context-map.md + architecture/ubiquitous-language.md
Conv 1b  Layer 3   architecture/architecture.md
Conv 2   Layer 3-4 architecture/event-catalog.md
Conv 3+  Layer 2   domains/<ctx>/use-cases.md             (one per context per PRD slice — needs domain-expert approval before next layer)
Conv 4+  Layer 3   domains/<ctx>/bounded-context.md       (one per context)
Conv 5+  Layer 3   domains/<ctx>/aggregates/<agg>.md      (one per aggregate)
Conv 6+  Layer 4   domains/<ctx>/api-contract.yaml + events.asyncapi.yaml + architecture/contracts/<a>-<b>.md
Conv 7+  Layer 6   architecture/adr/NNNN-...md            (as needed)
Conv 8   Layer 6   ai-rules/{development-rules,review-checklist,workflow-templates}.md
```

### What to Load in Each Conversation

| Generating... | Load First | Why |
|---------------|-----------|-----|
| Context Map | PRD + use-cases (if any) | Need full system view |
| Architecture | Context Map + PRD | Decisions must align with contexts |
| Ubiquitous Language (system) | Context Map | Terms must align with contexts |
| Event Catalog | Context Map + Glossary | Events cross context boundaries |
| Use Cases (Layer 2) | PRD slice + Context Map + Glossary | AI drafts; domain expert validates |
| Bounded Context | Use Cases (approved) + Context Map + Glossary + Event Catalog (summary) | Spec must trace to approved use cases |
| Aggregate | Parent context + Event Catalog (relevant events only) | Need boundaries and emitted events |
| OpenAPI / AsyncAPI | Bounded Context + Aggregates + Event Catalog | Wire format must mirror the model |
| Integration Contract (markdown) | Both context docs + YAMLs + Event Catalog | Need both sides |
| ADR | Triggering use case / PR + Dev Rules + Context Map | Decision must cite governance |
| AI Rules | Context Map + Architecture + sample of existing PRs | Rules must be enforceable in this repo |

### Context Window Management

**Rule: commit, push, and start a new conversation before context reaches 50%.**

Signs the context is getting full:
- Two or more large docs generated in one session.
- 20+ tool calls.
- Multiple Layer 3 docs loaded at once.

Per-conversation workflow:

```
1. Start fresh conversation
2. Load template + reference docs (Layer 1 docs)
3. Generate ONE doc (or 2-3 small related docs at the same layer)
4. Review the output
5. Commit & push
6. STOP — start new conversation for the next doc
```

If you must combine docs in one conversation:
- Same layer only (e.g., 2-3 small bounded contexts).
- Never mix layers (don't generate a context + its aggregates together).
- Commit after each doc, not at the end.
- If the AI starts repeating itself or losing track, stop immediately.

### Iterative Refinement

After the initial generation pass, refine in focused conversations:

```
Conv N:   "Review architecture/context-map.md — check that all contexts are
           listed and relationships match the per-domain bounded-context docs"

Conv N+1: "Review Related Docs links in all domains/*/bounded-context.md —
           ensure aggregate and contract paths are correct after the by-domain
           restructure"
```

Keep refinement conversations small and focused — one review task per conversation.
