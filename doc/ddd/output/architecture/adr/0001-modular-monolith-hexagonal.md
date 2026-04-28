# ADR-0001: Modular Monolith with Hexagonal Per-Bounded-Context Structure

**Status:** Proposed
**Date:** 2026-04-28
**Deciders:** Architecture team
**Consulted:** Quant / strategy author (PRD owner)
**Informed:** Implementers using `/implement`

## Context

The PRD (`doc/ddd/input/prd.md` §3, §11) describes an AI-driven pipeline that runs on a single host with one IBKR account, has strict separation of responsibilities across 8+ agent stages, and prefers parallel execution where possible. v1 has a single operator and a single broker connection (IBKR TWS/Gateway is not horizontally scalable per account). At the same time, the agent stages have very different change cadences — alpha frameworks are under active research, execution is regulatory-sensitive, and backtesting is read-only. We must pick a deployment style that supports DDD bounded-context isolation today without paying distributed-systems cost up front, while leaving a clear extraction path if any one context grows beyond the host.

## Decision

We will build the system as a **single Python process (modular monolith)** organized as one Python package per bounded context, each implemented in **hexagonal (ports & adapters)** style. Cross-context communication goes through an in-process `EventBus` port and explicit read-model interfaces — never through direct module imports across context boundaries. Each context owns its own domain model, repository implementations, and adapter layer.

## Scope

| Aspect | Value |
|--------|-------|
| Bounded Context(s) affected | All 10 contexts |
| Aggregates affected | All |
| Contracts affected | All Layer 4 contracts |
| Code modules affected | `src/signal_intelligence/*`, `src/portfolio_management/*`, `src/trade_operations/*`, `src/research_validation/*`, `src/core/*` |

## Options Considered

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | Microservices from day one (one service per BC) | Strongest runtime isolation; independent deploys | Overkill for single-host single-account v1; adds bus, discovery, ops cost; LLM/IBKR latency dominates anyway |
| 2 | Single flat package (no BC boundaries) | Fastest to write | Conflates research code with execution; no enforceable invariants; the PRD's "strict separation" cannot be honored |
| 3 | **Modular monolith with hexagonal per-BC structure (Chosen)** | Honors DDD boundaries; cheap to run; ports allow later extraction; enforceable via import-lint rules | Requires team discipline to not bypass ports; in-process bus means a crash takes the whole pipeline down |

## Consequences

**Positive**
- Bounded contexts are first-class Python packages with no cross-imports — the `Trading Loop` orchestrator is the only place sequencing lives.
- One `python -m` entrypoint; one `.venv`; matches CLAUDE.md's "always run in `.venv`" rule.
- Testing is simpler — invariant + contract tests run in-process without spinning up brokers.
- Extraction to services later is a port-swap, not a rewrite.

**Negative**
- A bug in one context can crash the whole loop. Mitigation: per-stage exception isolation in the orchestrator (ADR-0003).
- Single-process means no horizontal scaling. Acceptable for single-account IBKR.
- Discipline cost: contributors must not import `from signal_intelligence.alpha_generation` inside `trade_operations`. Enforced by import-linter in CI.

**Neutral / Follow-up**
- Add `import-linter` configuration listing forbidden cross-context imports.
- Define the `EventBus` port interface once, in `src/core/events/`.
- Decide later whether to migrate to PostgreSQL + a real broker (RabbitMQ/NATS) when v2 needs multi-process.

## Governance Impact

| Governance Rule | Effect of this ADR |
|-----------------|--------------------|
| HR-1 (no cross-context DB reads) | Conforms — enforced structurally because each BC owns its repository module. |
| HR-2 (shared kernel = value objects only) | Conforms — `src/core/` is restricted to VOs and the `EventBus` port. |
| HR-3 (contracts linted in CI) | Conforms — OpenAPI/AsyncAPI per BC fed into `spectral`. |

No deviation; no compensating control needed.

## Verification

- CI step runs `import-linter` with a contract forbidding cross-context imports. Build fails on violation.
- A smoke test starts the orchestrator with a mock IBKR adapter and verifies one full trading-loop pass executes through all stages.
- PR review checklist (`ai-rules/review-checklist.md`) includes a "no cross-context imports" item.

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [../context-map.md](../context-map.md) |
| Architecture | [../architecture.md](../architecture.md) |
| Development Rules | [../../ai-rules/development-rules.md](../../ai-rules/development-rules.md) |
| ADR-0002 (Shared Kernel scope) | [0002-configuration-shared-kernel.md](0002-configuration-shared-kernel.md) |
| ADR-0003 (Orchestrator + sync command) | [0003-orchestrator-and-sync-rebalance.md](0003-orchestrator-and-sync-rebalance.md) |
