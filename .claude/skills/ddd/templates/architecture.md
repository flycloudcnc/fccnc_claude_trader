# DDD Template: Architecture

## Usage

Use this template to document **cross-cutting technical architecture decisions** that span all bounded contexts. This covers the implementation patterns, infrastructure choices, and deployment topology that shape how the domain model is realized in code.

This is a **Layer 1** document — load it alongside the context map to understand both *what* the system does (context map) and *how* it's built (architecture).

---

## Template

```markdown
# Architecture

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

## Architectural Style

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Overall pattern | [e.g., Hexagonal / Layered / Clean / CQRS] | [Why this pattern] |
| Communication | [e.g., Event-driven / Request-response / Hybrid] | [Why this style] |
| Concurrency model | [e.g., Async / Threaded / Actor] | [Why this model] |
| State management | [e.g., Event sourcing / State-based / Hybrid] | [How state is persisted] |

## Tech Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Language | [e.g., Python] | [3.12] | [Primary language] |
| Framework | [e.g., FastAPI / Django / None] | [X.Y] | [Web/API framework] |
| Database | [e.g., PostgreSQL / MySQL] | [X.Y] | [Primary data store] |
| ORM | [e.g., SQLAlchemy] | [X.Y] | [Database abstraction] |
| Message bus | [e.g., RabbitMQ / In-process] | [X.Y] | [Event routing] |
| External APIs | [List key integrations] | — | [Data sources / services] |
| LLM | [e.g., Claude / GPT-4] | — | [AI decision-making] |
| ML | [e.g., scikit-learn] | [X.Y] | [Machine learning] |
| Testing | [e.g., pytest] | [X.Y] | [Test framework] |

## Infrastructure & Deployment

| Aspect | Choice | Details |
|--------|--------|---------|
| Runtime environment | [e.g., Docker / bare metal / serverless] | [Container setup] |
| Hosting | [e.g., AWS EC2 / GCP / self-hosted] | [Instance type, region] |
| Database hosting | [e.g., Docker container / managed RDS] | [Persistence strategy] |
| CI/CD | [e.g., GitHub Actions / Jenkins] | [Pipeline description] |
| Monitoring | [e.g., structlog + email alerts] | [Observability approach] |

## Project Structure

```text
[Directory tree showing how bounded contexts map to code modules]

project-root/
├── src/
│   ├── core/              <- Shared kernel (events, models)
│   ├── <context-a>/       <- Bounded context A
│   ├── <context-b>/       <- Bounded context B
│   └── engine/            <- Orchestration / infrastructure
├── data/                  <- Persistent data, models, configs
└── tests/                 <- Test suite
```

## Context-to-Code Mapping

| Bounded Context | Code Module(s) | Entry Point |
|----------------|----------------|-------------|
| [Context A] | `src/<module>/` | [Main class or function] |
| [Context B] | `src/<module>/` | [Main class or function] |

## Cross-Cutting Concerns

| Concern | Approach | Implementation |
|---------|----------|---------------|
| Logging | [e.g., Structured logging] | [Library, format, destination] |
| Error handling | [e.g., Domain exceptions + global handler] | [Strategy] |
| Configuration | [e.g., Environment variables + .env] | [Config loading approach] |
| Security | [e.g., API key management, secrets] | [How secrets are managed] |
| Scheduling | [e.g., APScheduler / cron] | [Task scheduling approach] |

## Data Flow Architecture

```text
[Diagram showing how data moves through the system at the infrastructure level]

External Source → ACL → Domain Events → Event Bus → Handlers → Database
```

## Key Architectural Constraints

1. [Constraint about performance, e.g., "Quote processing must complete within 50ms"]
2. [Constraint about reliability, e.g., "System must auto-reconnect within 60 seconds"]
3. [Constraint about cost, e.g., "LLM calls limited to entry decisions only"]
4. [Constraint about safety, e.g., "All parameter changes require human approval"]

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md] |
| Design Spec | [../design-spec-v2.md] |
| Ubiquitous Language | [ubiquitous-language.md] |
```

---

## Guidelines

1. **One architecture doc per system** — complements the context map with technical decisions.
2. **Focus on decisions, not descriptions** — every row should answer "what did we choose and why?"
3. **Keep it stable** — this doc changes rarely. If architectural decisions change frequently, the architecture isn't settled.
4. **Context-to-code mapping is critical** — this tells the AI agent exactly which code module implements which bounded context.
5. **Constraints are guardrails** — AI agents must respect these when implementing features.
6. **Layer 1 doc** — load alongside context-map.md for full system orientation. Links to all Layer 2 docs via context-map's Related Docs.
7. **Target ~120 lines** — keep concise. Detailed infrastructure setup belongs in deployment docs, not here.
