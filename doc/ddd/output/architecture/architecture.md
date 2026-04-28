# Architecture

**Version:** 0.1
**Last Updated:** 2026-04-28

## Architectural Style

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Overall pattern | Hexagonal (ports & adapters) per bounded context, DDD-aligned modular monolith | Each BC keeps its own domain model and ports; allows extraction to services later without rewrite. |
| Communication | Hybrid — async events between most contexts, sync read-models for large data, sync command for `rebalancing → execution` | Async decouples the signal pipeline; sync command lets the rebalancer fail-fast on broker rejection (ADR-0003). |
| Concurrency model | Async (Python `asyncio`) with a single-process orchestrator; per-context handlers run as awaited tasks | Matches the LLM I/O-bound workload and the IBKR API's single-connection model; avoids multi-process complexity in v1. |
| State management | State-based persistence per aggregate (no event sourcing in v1); domain events emitted for integration only | Simpler operational model; event log is a side-effect, not the source of truth. Revisit if audit/replay becomes a hard requirement. |
| Coordination | Central orchestrator (`Trading Loop`) drives the pipeline on a configurable cadence; contexts also react to events (ADR-0003) | Deterministic ordering for the daily/intra-day pass; events handle reactive updates between passes. |
| Agent style | LLM-driven agents per investment framework, returning strict JSON parsed into domain value objects (ADR-0005) | Frameworks in `investment-frameworks.md` are written as Claude prompts; pluggable strategy pattern in `alpha-generation` (ADR pending). |

## Tech Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Language | Python | 3.12+ | Primary language (matches `.venv` convention from CLAUDE.md). |
| Framework | None (CLI + scheduled jobs) — FastAPI only if a UI/API is added later | — | Internal pipeline; no external HTTP surface in v1. |
| Database | SQLite (v1) → PostgreSQL (v2 if multi-process) | 3.x / 16.x | Persist snapshots, signals, targets, orders, positions. |
| ORM | SQLAlchemy 2.x | 2.x | Repository implementation per aggregate. |
| Message bus | In-process pub/sub (Python `asyncio.Queue` wrapper) | — | Per ADR-0003: in-process is sufficient for monolith; abstract behind `EventBus` port so it can be swapped for RabbitMQ/NATS later. |
| External APIs | IBKR TWS / Client Portal Gateway; SEC EDGAR (insider Form 4, 13F-HR); price/fundamentals vendor TBD | — | IBKR for execution; EDGAR direct (per decision #11). |
| LLM | Claude (Anthropic API) | claude-opus-4-7 / claude-sonnet-4-6 (per task) | Agent reasoning; strict JSON output per ADR-0005. Use Sonnet for high-volume screening, Opus for synthesis. |
| ML | None in v1 | — | Pure rule + LLM scoring; ML deferred. |
| Testing | pytest, pytest-asyncio, hypothesis (for invariants) | latest | Unit + invariant + contract tests. |
| Contract lint | spectral | latest | Lint OpenAPI 3.1 + AsyncAPI 2.6 in CI (HR-3). |
| Logging | structlog | latest | JSON log records keyed by `run_id`, `ticker`, `context`. |
| Scheduling | APScheduler | latest | Drives the configurable trading-loop cadence (decision #7). |

## Infrastructure & Deployment

| Aspect | Choice | Details |
|--------|--------|---------|
| Runtime environment | Local `.venv` (dev) → Docker (single image, multi-process via `supervisord`) for staging/prod | Per CLAUDE.md: always run in `.venv` during development. |
| Hosting | Self-hosted single VM (v1); IBKR TWS/Gateway on the same host | Single-account, USD-only constraint (decisions #9, #12) keeps topology simple. |
| Database hosting | SQLite file in `data/` (v1); managed Postgres (v2) | Backup the SQLite file on every successful trading-loop pass. |
| CI/CD | GitHub Actions | Lint (`ruff`), tests, contract lint (`spectral`), build, push image. |
| Monitoring | structlog → file + (optional) email/Slack alert sink for `RiskAlertRaised` and uncaught exceptions | Trading-loop heartbeat written every cycle. |
| Secrets | `.env` (dev) + OS keyring or HashiCorp Vault (prod) for IBKR credentials and Anthropic API key | Never commit secrets; loaded once at process start. |

## Project Structure

```text
project-root/
├── src/
│   ├── core/                                  <- Shared kernel (Configuration VOs, primitives, EventBus port)
│   │   ├── config/                            <- RiskTolerance, TargetReturn, MaxDrawdown, Horizon, PortfolioSize
│   │   ├── primitives/                        <- Ticker, Money, Weight, Percent
│   │   └── events/                            <- EventBus port + base Event class
│   ├── signal_intelligence/
│   │   ├── market_data/                       <- BC: market-data
│   │   ├── alpha_generation/                  <- BC: alpha-generation (with frameworks/ subpackage of pluggable strategies)
│   │   ├── conviction_sizing/                 <- BC: conviction-sizing
│   │   └── catalyst_timing/                   <- BC: catalyst-timing
│   ├── portfolio_management/
│   │   ├── risk_regime/                       <- BC: risk-regime
│   │   ├── portfolio_construction/            <- BC: portfolio-construction
│   │   └── rebalancing/                       <- BC: rebalancing
│   ├── trade_operations/
│   │   ├── execution/                         <- BC: execution (IBKR adapter is here)
│   │   └── monitoring/                        <- BC: monitoring
│   ├── research_validation/
│   │   └── backtesting/                       <- BC: backtesting
│   ├── orchestrator/                          <- Trading Loop application service (no domain logic)
│   └── adapters/                              <- LLM client, IBKR client, EDGAR client, vendor price client (infra)
├── data/                                      <- SQLite, downloaded EDGAR filings cache
├── doc/                                       <- prd.md, design-spec, ddd output
└── tests/
    ├── unit/                                  <- Per-context unit tests
    ├── invariants/                            <- HR-4 invariant tests
    └── contract/                              <- HR-3 contract tests against api-contract.yaml / events.asyncapi.yaml
```

## Context-to-Code Mapping

| Bounded Context | Code Module(s) | Entry Point |
|----------------|----------------|-------------|
| `signal-intelligence/market-data` | `src/signal_intelligence/market_data/` | `MarketDataService.ingest_pass()` |
| `signal-intelligence/alpha-generation` | `src/signal_intelligence/alpha_generation/` | `AlphaScreenerService.run_pass(config)` |
| `signal-intelligence/conviction-sizing` | `src/signal_intelligence/conviction_sizing/` | `ConvictionService.assess(candidate)` |
| `signal-intelligence/catalyst-timing` | `src/signal_intelligence/catalyst_timing/` | `CatalystService.evaluate(candidate)` |
| `portfolio-management/risk-regime` | `src/portfolio_management/risk_regime/` | `RiskRegimeService.recompute()` |
| `portfolio-management/portfolio-construction` | `src/portfolio_management/portfolio_construction/` | `PortfolioConstructionService.compose(signals, regime)` |
| `portfolio-management/rebalancing` | `src/portfolio_management/rebalancing/` | `RebalanceService.evaluate(target)` |
| `trade-operations/execution` | `src/trade_operations/execution/` | `ExecutionService.submit(plan)` |
| `trade-operations/monitoring` | `src/trade_operations/monitoring/` | `MonitoringService.snapshot()` |
| `research-validation/backtesting` | `src/research_validation/backtesting/` | `BacktestRunner.run(strategy, window)` |
| Trading Loop orchestrator | `src/orchestrator/` | `TradingLoop.run()` |

## Cross-Cutting Concerns

| Concern | Approach | Implementation |
|---------|----------|---------------|
| Logging | Structured JSON via `structlog`; mandatory keys `run_id`, `context`, `ticker` (where applicable). | One logger per context module; orchestrator stamps `run_id`. |
| Error handling | Domain exceptions per context (`AlphaScreeningError`, `IbkrSubmitError`, etc.) caught by the orchestrator; uncaught = process restart by supervisor. | No silent fallbacks. Broker failures bubble up synchronously per ADR-0003. |
| Configuration | `Configuration` shared kernel value objects loaded at process start from `config/portfolio.toml` + `.env`. | Validated via `pydantic` at load; immutable thereafter (ADR-0002). |
| LLM call discipline | Each agent uses a documented prompt template with strict JSON output schema; retries on parse failure (max 2); failures recorded as `AgentParseError`. | Prompts stored under `src/<ctx>/prompts/`; outputs validated against `pydantic` models before becoming domain VOs (ADR-0005). |
| Security | IBKR credentials never logged; Anthropic API key in env only; SEC EDGAR requires no auth (rate-limit aware). | `.env.example` checked in; secrets excluded by `.gitignore`. |
| Scheduling | `APScheduler` cron job in the orchestrator drives the `TradingLoop.run()` at the configured cadence (intra-day, daily, weekly — chosen at deploy). | Cadence in `Configuration`; orchestrator validates against market hours via a calendar adapter. |
| Idempotency | Every event carries a `(event_name, aggregate_id, version)` key; subscribers de-dup on first write. | Enforced by `EventBus` wrapper; HR-7. |
| Audit | Every LLM invocation + IBKR order is logged to an append-only `audit/` jsonl with model + prompt-hash + response-hash. | Required by AI Workflow Templates' Invocation Audit. |

## Data Flow Architecture

```text
External Sources                    market-data ACL                Domain
─────────────────                   ───────────────                ──────
IBKR TWS API ─────► IbkrClient ───► (no — execution writes only)
SEC EDGAR    ─────► EdgarClient ──► XBRL parser ──► InsiderTransaction / ThirteenFFiling
Price Vendor ─────► VendorClient ─► Normalizer ───► PriceSnapshot / FundamentalSnapshot
                                                          │
                                                          ▼
                                           SQLite (per-context schemas)
                                                          │
                                          ┌───────────────┼───────────────┐
                                          ▼               ▼               ▼
                                   Sync read-models   Domain Events    Repositories
                                   (snapshot query)   (asyncio bus)    (per aggregate)
                                          │               │
                                          ▼               ▼
                          alpha / conviction / catalyst / risk-regime
                                          │
                                          ▼
                                portfolio-construction
                                          │
                                          ▼
                                    rebalancing  ──(sync command)──► execution ──► IBKR
                                                                         │
                                                                         ▼
                                                                   monitoring
```

## Key Architectural Constraints

1. **Latency is not critical**, but the full trading-loop pass must complete within the configured cadence interval (e.g., < 5 minutes for an intra-day cadence). The orchestrator emits a `LoopOverrun` log event if a pass exceeds 80% of the interval.
2. **Reliability** — the orchestrator must auto-resume from the last committed state after a process restart; in-flight events are re-published on startup based on the persisted aggregate version.
3. **Cost discipline** — LLM calls are the dominant cost driver. Use Claude Sonnet for high-volume screening (`alpha-generation`) and Claude Opus only for synthesis steps (`conviction-sizing`, `catalyst-timing` reasoning chains). Cache framework prompts via the Anthropic prompt cache.
4. **Safety** — all live order submission is gated by a `paper_trading` flag (ADR-0004). Flipping it to `false` requires a human-approved config change and an ADR.
5. **Strict JSON only** — LLM-driven agents must return parseable JSON conforming to a published schema (ADR-0005). Free-text replies are a hard error.
6. **Single IBKR account, USD-only** in v1 (decisions #9, #12). Multi-account or multi-currency requires an ADR and changes to `Order`, `AccountPosition`, and `PortfolioHealthSnapshot` aggregates.
7. **No code outside a documented bounded context.** Adding a new module requires updating `context-map.md` and this doc in the same PR.

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md](context-map.md) |
| Ubiquitous Language | [ubiquitous-language.md](ubiquitous-language.md) |
| Event Catalog | [event-catalog.md](event-catalog.md) |
| ADR Index | [adr/](adr/) |
| AI Development Rules | [../ai-rules/development-rules.md](../ai-rules/development-rules.md) |
| PRD | [../../doc/ddd/input/prd.md](../../doc/ddd/input/prd.md) |
| Investment Frameworks | [../../doc/ddd/input/investment-frameworks.md](../../doc/ddd/input/investment-frameworks.md) |
