# Context Map

**Version:** 0.1
**Last Updated:** 2026-04-28

## Bounded Contexts Overview

| Context | Purpose | Upstream Of | Downstream Of |
|---------|---------|------------|---------------|
| `signal-intelligence/market-data` | Ingest & normalize prices, fundamentals, insider transactions, 13F filings from external sources (SEC EDGAR for insider/13F). | `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`, `monitoring`, `backtesting` | (External data sources) |
| `signal-intelligence/alpha-generation` | Run pluggable framework strategies (Greenblatt, Goldman, Lynch, Mayer, Wood, Camillo, Insider/13F, Blind-Spots) to produce ranked candidate tickers with alpha scores. | `conviction-sizing`, `catalyst-timing`, `portfolio-construction`, `backtesting` | `market-data`, `configuration` (SK) |
| `signal-intelligence/conviction-sizing` | Grade per-candidate conviction and recommend a target weight range (Pabrai / Mayer / Sleep frameworks). | `portfolio-construction`, `rebalancing`, `backtesting` | `alpha-generation`, `market-data` |
| `signal-intelligence/catalyst-timing` | Identify catalysts and recommend an entry-timing action per candidate (Burry / Marks / Insider Buying frameworks). | `portfolio-construction`, `rebalancing`, `backtesting` | `alpha-generation`, `market-data` |
| `portfolio-management/risk-regime` | Determine the prevailing portfolio-level exposure regime and publish regime changes (Marks / Pabrai / Burry frameworks). | `portfolio-construction`, `rebalancing` | `market-data`, `monitoring`, `configuration` (SK) |
| `portfolio-management/portfolio-construction` | Combine alpha + conviction + catalyst signals with the risk regime and configured constraints into target portfolio weights. | `rebalancing`, `backtesting` | `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`, `configuration` (SK) |
| `portfolio-management/rebalancing` | Detect drift / event triggers against a target portfolio and emit a rebalance plan. | `execution` | `portfolio-construction`, `risk-regime`, `monitoring` |
| `trade-operations/execution` | Submit, confirm, and reconcile orders against a single IBKR account (paper-trading first); maintain `AccountPosition` state. | `monitoring`, `portfolio-construction` (positions read-model) | `rebalancing` |
| `trade-operations/monitoring` | Compute portfolio health snapshots and raise risk alerts; feedback loop into `risk-regime` and `rebalancing`. | `risk-regime`, `rebalancing` | `execution`, `market-data` |
| `research-validation/backtesting` | Replay historical data through the signal + portfolio pipeline to evaluate strategies. Read-only; no live execution. | (Researcher / orchestrator) | `market-data`, `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `portfolio-construction` |

Domains: `signal-intelligence`, `portfolio-management`, `trade-operations`, `research-validation`. The cross-cutting **`Configuration` shared kernel** (value objects: `RiskTolerance`, `TargetReturn`, `MaxDrawdown`, `Horizon`, `PortfolioSize`) is owned by the architecture team and consumed by 5+ contexts — see ADR-0002.

The **`Trading Loop` orchestrator** is an application service (not a bounded context) that drives the pipeline on a configurable cadence — see ADR-0003 and `architecture.md`.

## Relationship Types

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
| (External: IEX/Polygon, fundamentals vendor) | `market-data` | ACL | REST polling | Anti-corruption against vendor schemas. |
| (External: SEC EDGAR) | `market-data` | ACL | HTTP fetch + XBRL parse | Direct EDGAR fetch for insider Form 4 and 13F filings. |
| `market-data` | `alpha-generation` | OHS / PL | Sync read-model (snapshot query) | Universe + fundamentals snapshots; large per-pass dataset. |
| `market-data` | `conviction-sizing` | OHS / PL | Sync read-model | Per-ticker fundamentals + price history. |
| `market-data` | `catalyst-timing` | OHS / PL | Sync read-model + event subscription | Reads insider clusters, earnings dates; subscribes to `InsiderTransactionRecorded`. |
| `market-data` | `risk-regime` | OHS / PL | Sync read-model | Index/breadth/volatility series. |
| `market-data` | `monitoring` | OHS / PL | Sync read-model | Latest prices for portfolio valuation. |
| `market-data` | `backtesting` | OHS / PL | Sync historical query | Time-bounded historical replay. |
| `alpha-generation` | `conviction-sizing` | PL | Async event: `AlphaCandidateGenerated` | New candidate triggers conviction grading. |
| `alpha-generation` | `catalyst-timing` | PL | Async event: `AlphaCandidateGenerated` | New candidate triggers catalyst scan. |
| `conviction-sizing` | `portfolio-construction` | PL | Async event: `ConvictionAssessed` | Conviction signals fed into target-weight composition. |
| `catalyst-timing` | `portfolio-construction` | PL | Async event: `CatalystIdentified` | Catalyst signals fed into target-weight composition. |
| `alpha-generation` | `portfolio-construction` | PL | Async event: `AlphaCandidateGenerated` | Alpha scores fed into target-weight composition. |
| `risk-regime` | `portfolio-construction` | PL | Async event: `RiskRegimeChanged` + sync read | Regime constrains exposure caps. |
| `risk-regime` | `rebalancing` | PL | Async event: `RiskRegimeChanged` | Regime change triggers rebalance evaluation. |
| `portfolio-construction` | `rebalancing` | PL | Async event: `PortfolioTargetGenerated` | New target triggers drift check. |
| `rebalancing` | `execution` | PL / CS | Sync command (`SubmitRebalancePlan`) | Synchronous so the rebalancer can fail fast on broker rejection. |
| `execution` | `monitoring` | PL | Async events: `OrderFilled`, `PositionUpdated` | Live state feeds health metrics. |
| `execution` | `portfolio-construction` | OHS | Sync read-model: `AccountPositionView` | Construction reads current positions to compute deltas. |
| `monitoring` | `risk-regime` | PL | Async event: `RiskAlertRaised` | Health alerts can shift regime. |
| `monitoring` | `rebalancing` | PL | Async event: `RiskAlertRaised` | Health alerts can trigger rebalance. |
| `Configuration` (SK) | `alpha-generation`, `conviction-sizing`, `risk-regime`, `portfolio-construction`, `rebalancing`, `monitoring` | SK | Shared module of value objects | See ADR-0002. |

## Data Flow Diagram

```text
                          (External)
   IEX / Polygon ──┐                            ┌── SEC EDGAR (insider, 13F)
                   ▼                            ▼
              ┌──────────────────────────────────────────┐
              │     signal-intelligence/market-data       │
              └──────────────────────────────────────────┘
                   │              │              │
                   │ snapshot     │ snapshot     │ snapshot
                   ▼              ▼              ▼
   ┌──────────── alpha-generation ─────────────┐
   │                                            │
   │ AlphaCandidateGenerated                    │
   ▼                                            ▼
conviction-sizing                       catalyst-timing
   │ ConvictionAssessed                         │ CatalystIdentified
   └────────────────┐    ┌───────────────────────
                    ▼    ▼
            ┌───────────────────────────┐    RiskRegimeChanged
            │  portfolio-construction   │  ◄────── risk-regime ◄── market-data
            └───────────────────────────┘
                       │ PortfolioTargetGenerated
                       ▼
                ┌──────────────┐
                │ rebalancing  │ ◄─── RiskAlertRaised ─── monitoring
                └──────────────┘
                       │ SubmitRebalancePlan (sync)
                       ▼
                ┌──────────────┐  OrderFilled / PositionUpdated
                │  execution   │ ─────────────────────────► monitoring
                │   (IBKR)     │ ◄── AccountPositionView (read) ── portfolio-construction
                └──────────────┘

   research-validation/backtesting reads from market-data, alpha-generation,
   conviction-sizing, catalyst-timing, portfolio-construction (no writes
   into live contexts; replays historical data through the same pipeline).
```

## Dependency Rules

1. `execution` is the only context allowed to call the IBKR API. All other contexts that need live broker state read it via `AccountPositionView`.
2. `market-data` publishes events and exposes read-models; it never consumes events from other contexts.
3. No context in `portfolio-management` or `trade-operations` may import from `signal-intelligence` modules; they communicate only via the documented events / read-models.
4. `research-validation/backtesting` is read-only against live contexts. It must never emit live commands or events on the production bus (must publish to a dedicated `backtest.*` topic or in-memory bus).
5. The `Configuration` shared kernel contains **value objects only** (no behaviour, no repositories) per HR-2.
6. The `Trading Loop` orchestrator may read from any context but may not implement domain logic — only sequencing, scheduling, and error escalation.
7. Cross-context reads of another context's database tables are forbidden (HR-1). All cross-context reads go through APIs, events, or materialized views.
8. Every event listed here must appear in `event-catalog.md` and the publishing context's `events.asyncapi.yaml`.

## Shared Models (if any)

| Model | Owned By | Shared With | Sharing Mechanism |
|-------|---------|-------------|-------------------|
| `RiskTolerance`, `TargetReturn`, `MaxDrawdown`, `Horizon`, `PortfolioSize` (Configuration) | Architecture team | `alpha-generation`, `conviction-sizing`, `risk-regime`, `portfolio-construction`, `rebalancing`, `monitoring` | Shared Kernel (see ADR-0002) |
| `Ticker`, `Money`, `Weight`, `Percent` (primitive value objects) | Architecture team | All contexts | Shared Kernel (see ADR-0002) |

## Anti-Corruption Layers

| ACL Location | Upstream Context | Translation |
|-------------|-----------------|-------------|
| `market-data` ingestion adapters | External price/fundamentals vendors | Vendor JSON → `PriceSnapshot` / `FundamentalSnapshot` value objects with normalized fields. |
| `market-data` EDGAR adapter | SEC EDGAR (Form 4, 13F-HR) | XBRL / SGML → `InsiderTransaction` / `ThirteenFFiling` aggregates; cluster detection lives downstream in `catalyst-timing`. |
| `execution` IBKR adapter | IBKR TWS / Client Portal API | IBKR order/contract objects → `Order` / `Fill` / `AccountPosition` aggregates; isolates IBKR field naming + error codes. |

## Related Docs

| Doc | Path |
|-----|------|
| Architecture | [architecture.md](architecture.md) |
| Ubiquitous Language | [ubiquitous-language.md](ubiquitous-language.md) |
| Event Catalog | [event-catalog.md](event-catalog.md) |
| ADR Index | [adr/](adr/) |
| AI Development Rules | [../ai-rules/development-rules.md](../ai-rules/development-rules.md) |
