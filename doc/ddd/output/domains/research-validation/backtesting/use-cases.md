# Intent Translation: research-validation/backtesting

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§6 Data Requirements; §8 Phase 3; §9 Critical Risks; §12 Next Step #3)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Replay historical market-data through the same signal-intelligence + portfolio-management pipeline (alpha-generation → conviction-sizing → catalyst-timing → risk-regime → portfolio-construction) over a fixed `BacktestWindow` with a fixed `BacktestStrategy` (the configured framework set + constraints + regime model), and produce a `BacktestRun` with summary metrics (CAGR, max drawdown, Sharpe, hit rate). **Read-only against live contexts; never emits live commands or events on the production bus** (context-map dependency rule #4) — backtests publish to a dedicated `backtest.*` topic or in-memory bus.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `research-validation/backtesting` | Yes | Owns `Backtest`, `BacktestRun` aggregates and the backtest event bus. |
| `signal-intelligence/market-data` | No | Sync historical query for `PriceSnapshot`, `FundamentalSnapshot`, `InsiderTransaction`, `ThirteenFFiling` over the window. |
| `signal-intelligence/alpha-generation` | No | Re-invoked in backtest mode (sandboxed bus) per simulated cadence tick. |
| `signal-intelligence/conviction-sizing` | No | Re-invoked in backtest mode. |
| `signal-intelligence/catalyst-timing` | No | Re-invoked in backtest mode. |
| `portfolio-management/portfolio-construction` | No | Re-invoked in backtest mode. |
| `portfolio-management/risk-regime` | No | Re-invoked in backtest mode (regime computed from historical breadth/volatility). |

> Note: `rebalancing`, `execution`, and `monitoring` are **not** invoked in v1 backtests. The backtest directly evaluates `PortfolioTarget`s against historical prices to produce hypothetical returns; broker / fill simulation is deferred. This bound matches PRD §8 Phase 3 + §12 Next Step #3 framing of backtesting as a research harness, not an execution simulator.

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Backtesting system (PRD §8 Phase 3) | BacktestRunner application service | `BacktestRunner` | Existing (per architecture.md context-to-code map) |
| Backtest run (researcher artefact) | BacktestRun aggregate | `BacktestRun` | Existing |
| Backtest window (researcher input) | BacktestWindow VO `[start_date, end_date]` | `BacktestWindow` | Existing |
| Strategy under test (configured framework set + constraints + regime model) | BacktestStrategy VO | `BacktestStrategy` | Existing |
| Strategy hash (audit / dedupe key) | strategy_hash (event payload field) | `strategy_hash: string` | Existing |
| Summary metrics (CAGR, max DD, Sharpe, hit rate) | summary_metrics (event payload field) | `summary_metrics: object` | Existing |
| Backtest pass-equivalent of `RunId` | BacktestRunId (per simulated cadence tick) | `BacktestRunId` | New (analogue of `RunId` scoped to the backtest) |
| Backtest bus | dedicated `backtest.*` topic / in-memory bus | `BacktestEventBus` | New (anchor: context-map rule #4) |

## Use Cases

### UC-1: ConfigureAndStartBacktest

- **Actor:** Researcher (CLI / notebook).
- **Trigger:** `BacktestRunner.run(strategy, window)` invoked.
- **Goal:** Validate inputs, allocate per-backtest scratch state (per-backtest scratch DB per event-catalog `PriceSnapshotIngested` failure-handling row), instantiate a sandboxed `BacktestEventBus`, and queue simulated cadence ticks across the window.
- **Primary Aggregate:** `BacktestRun`.
- **Pre-conditions:** Historical data available for the entire `BacktestWindow` (else fail-fast — partial-data backtests are misleading); `strategy_hash` computed; researcher has provided seed for any stochastic component (none in v1, but reserved).
- **Outcome (success):** `BacktestRun(status=in_progress)` persisted with `backtest_id`, `strategy_hash`, `window_start`, `window_end`; cadence-tick queue populated; UC-2 begins consuming.
- **Outcome (failure):** Missing data for any date in window → `BacktestRun(status=failed, reason=DataGap)`; emit `BacktestCompleted(status=failed)`; do not proceed. Strategy validation failure (e.g., an enabled framework has no prompt) → fail-fast at config time, no `BacktestRun` persisted.
- **Invariants Touched:** `window_start ≤ window_end`; both must be ≤ today − 1 (no peeking at future / today's incomplete data); `strategy_hash` deterministic over `BacktestStrategy` content (different config = different hash).
- **Cross-context Calls:** Sync historical query to `market-data` (per integration-contract `market-data-backtesting.md`); allocates `BacktestEventBus`.

### UC-2: ReplayCadenceTick

- **Actor:** `BacktestRunner` (internal loop).
- **Trigger:** For each simulated cadence tick `t` in `[window_start, window_end]` at the strategy's cadence interval.
- **Goal:** For tick `t`: load the historical `Universe` + `FundamentalsView` + `InsiderClusterView` + `LatestPriceView` **as-of** `t` (point-in-time correctness — no look-ahead); invoke `risk-regime` → `alpha-generation` → `conviction-sizing` → `catalyst-timing` → `portfolio-construction` on the `BacktestEventBus`; capture the resulting `PortfolioTarget` for tick `t`.
- **Primary Aggregate:** `BacktestRun` (appends per-tick `PortfolioTarget` snapshot).
- **Pre-conditions:** Cadence tick `t` has the required as-of historical reads available.
- **Outcome (success):** `PortfolioTarget` for `t` persisted in scratch DB; downstream `rebalancing`/`execution` are NOT invoked in v1.
- **Outcome (failure):** A signal-intelligence framework's LLM call fails after retries → that framework contributes 0 for the tick (matches `alpha-generation` UC-1 failure semantics); the backtest continues. Per-backtest scratch DB write failure → fail backtest run (event-catalog `PriceSnapshotIngested` failure-handling row).
- **Invariants Touched:** **Point-in-time correctness** — every read (price, fundamentals, insider, 13F) MUST be as-of `t`, never later (no look-ahead). Backtest events MUST flow on the `BacktestEventBus` only — never the production bus (context-map rule #4). `BacktestRunId` correlates events for one tick within one backtest. Determinism on `(strategy_hash, window, seed)` — same inputs reproduce the same tick-by-tick targets (LLM `temperature=0` + prompt cache).
- **Cross-context Calls:** Historical reads from `market-data` (sync); re-invokes signal + portfolio contexts on the sandboxed bus.

### UC-3: SimulateReturnsFromTargets

- **Actor:** `BacktestRunner`.
- **Trigger:** After all cadence ticks are processed in UC-2.
- **Goal:** Walk the per-tick `PortfolioTarget` series; for each interval `[t, t+Δ]` compute the hypothetical portfolio return as `Σ target_weight_i,t × ((price_i,t+Δ − price_i,t) / price_i,t) + (cash_target_t × cash_return_t)`; account for transaction costs via a configured slippage + commission model (default v1: zero — see open question).
- **Primary Aggregate:** `BacktestRun`.
- **Pre-conditions:** Per-tick `PortfolioTarget` series complete; price series available across all intervals.
- **Outcome (success):** Per-interval returns persisted; portfolio NAV path computed; ready for UC-4.
- **Outcome (failure):** Missing price for a held ticker mid-interval → carry-forward last price; flag the backtest as `partial_pricing` in audit. If > N% of intervals affected, fail backtest.
- **Invariants Touched:** Returns computed on **target** weights (v1 simplification — does NOT model fill slippage / partial fills since execution is not invoked); v1 cash-return defaults to 0 unless a yield curve is provided.
- **Cross-context Calls:** None — pure computation over already-loaded data.

### UC-4: ComputeSummaryMetricsAndComplete

- **Actor:** `BacktestRunner`.
- **Trigger:** UC-3 completes.
- **Goal:** Compute `summary_metrics` — CAGR, max drawdown, annualized volatility, Sharpe (vs. configurable risk-free rate), hit rate (% of intervals positive), best/worst interval; persist to `BacktestRun.summary_metrics`; emit `BacktestCompleted(status=success)` on the backtest bus / researcher channel.
- **Primary Aggregate:** `BacktestRun(status=success)`.
- **Pre-conditions:** UC-3 returns series complete.
- **Outcome (success):** `BacktestRun` finalized; `BacktestResultsView` projection refreshed; researcher reads via CLI / notebook.
- **Outcome (failure):** Metric computation overflow / NaN (e.g., volatility on zero variance) → metric set to null with reason; do not fail the run.
- **Invariants Touched:** `summary_metrics` keys are stable (CAGR, max_drawdown, sharpe, volatility, hit_rate); `BacktestCompleted.status ∈ {success, failed}` (event-catalog enum).
- **Cross-context Calls:** Emits `BacktestCompleted` (researcher-facing only per event-catalog).

### UC-5: CompareTwoBacktests

- **Actor:** Researcher (CLI / notebook).
- **Trigger:** `BacktestRunner.compare(backtest_id_a, backtest_id_b)`.
- **Goal:** Produce a side-by-side comparison report (per-metric delta + visual NAV path overlay handled by researcher tooling; backtesting BC produces the structured comparison data, not the visualization).
- **Primary Aggregate:** N/A — read-only over two finalized `BacktestRun`s.
- **Pre-conditions:** Both runs have `status=success` and the same `BacktestWindow` (or compatible — comparing different windows yields a warning only, not a failure).
- **Outcome (success):** Comparison object returned with per-metric deltas and a sanity flag if `strategy_hash` is identical (suspicious comparison).
- **Outcome (failure):** One or both runs not finalized → error; researcher must wait or re-run.
- **Invariants Touched:** No new aggregate state; this is a query.
- **Cross-context Calls:** Reads `BacktestResultsView`.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `BacktestWindow.start ≤ end ≤ today − 1` | `BacktestRun` | New (no look-ahead at config time) |
| Point-in-time correctness — every read in UC-2 is as-of the simulated tick `t` | `BacktestRunner` | New (anchor: context-map rule #4) |
| Backtest events flow on `BacktestEventBus` only — never production bus | `BacktestRunner` | New (anchor: context-map rule #4) |
| `strategy_hash` deterministic over `BacktestStrategy` content | `BacktestRun` | New |
| Determinism on `(strategy_hash, window, seed)` — same inputs reproduce the same per-tick `PortfolioTarget` (LLM `temperature=0`, prompt cache) | `BacktestRunner` | New |
| `rebalancing` and `execution` are NOT invoked in v1 (target-weight simulation only) | `BacktestRunner` | New (architectural — v1 scope) |
| `BacktestCompleted.status ∈ {success, failed}` (event-catalog enum) | `BacktestRun` | New |
| Per-backtest scratch DB is isolated; cleanup on completion (success or failed) | `BacktestRunner` | New |
| Missing data over the window is fail-fast (UC-1) — partial-data backtests are misleading | `BacktestRunner` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `market-data` | Sync historical query (per `market-data-backtesting.md`) | Time-bounded replay over `BacktestWindow`. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `RiskTolerance`, `MaxDrawdown`, `Horizon`, `PortfolioSize` — strategy parameters. |
| Outbound | `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`, `portfolio-construction` | In-process re-invocation on `BacktestEventBus` | Re-runs the pipeline per simulated tick. |
| Outbound | (Researcher channel) | Event: `BacktestCompleted` (low-criticality, researcher-facing) | Run finalization signal. |
| Outbound | (Researcher channel) | Read-model: `BacktestResultsView` | UI / notebook consumption. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| `BacktestRunner` orchestration | `backtesting` | TBD |
| Sandboxed `BacktestEventBus` implementation | `backtesting` × architecture | TBD |
| Point-in-time historical data adapter (over `market-data` schema) | `backtesting` | TBD |
| Returns + metrics calculator (UC-3, UC-4) | `backtesting` | TBD |
| Strategy hash function | `backtesting` | TBD |

## Constraints & Prohibited Actions

1. **Read-only against live contexts** — no live commands, no live event publishing on the production bus (context-map dependency rule #4). All bus traffic stays on `BacktestEventBus`.
2. **No live order writes** — `execution` is not invoked; `IbkrClient` is not instantiated for backtests.
3. **Point-in-time correctness** — no look-ahead. Every per-tick read must use the as-of historical view; reads accidentally returning today's value are a hard correctness bug.
4. **`status=failed` on data gaps** — fail-fast in UC-1; do not silently produce a backtest over partial data.
5. **Determinism** — same `(strategy_hash, window, seed)` MUST reproduce identical per-tick `PortfolioTarget` and identical `summary_metrics`. LLM agents use `temperature=0` and Anthropic prompt cache (architecture.md §Key Constraints #3).
6. **Per-backtest isolation** — per-backtest scratch DB is created on UC-1, dropped on completion; never share scratch state across runs.
7. **No execution simulation in v1** — this is explicit scope: v1 evaluates target weights against historical prices. Slippage / partial-fill modelling is deferred to a Phase 2 enhancement (PRD §8 Phase 3 framing of backtesting as a research harness).
8. **Cost discipline** — long backtests can spend significant LLM tokens; prompt-cache aggressively, batch where possible, and warn the researcher of estimated cost before UC-1 begins (TBD with domain expert).

## Acceptance Criteria (testable)

1. Given a `BacktestStrategy` and a `BacktestWindow=[2023-01-01, 2023-12-31]` with full historical data, when `ConfigureAndStartBacktest` is invoked, then a `BacktestRun(status=in_progress)` is persisted with `strategy_hash` matching the strategy content.
2. Given the same strategy and window are submitted twice, when both run to completion, then `summary_metrics` are identical (deterministic on `(strategy_hash, window, seed)`).
3. Given the window includes a date with no `PriceSnapshot` for ≥ 1 universe ticker, when `ConfigureAndStartBacktest` validates, then it fails fast with `status=failed, reason=DataGap`, and `BacktestCompleted(status=failed)` is emitted.
4. Given a backtest is running, when any framework attempts to publish to the production bus, then the publish is rejected (sandboxed bus only); a hard correctness error is logged.
5. Given the backtest reads `FundamentalsView` for a ticker as-of `t`, when the backing data has a later filing dated `t + 30 days`, then the as-of read MUST return the prior filing (point-in-time correctness — no look-ahead).
6. Given UC-2 produces 252 daily `PortfolioTarget` snapshots over a 1-year window, when UC-3 runs, then the NAV path has 252 intervals and the per-interval return uses target weights at `t` and prices at `t` and `t+Δ`.
7. Given UC-3 completes, when UC-4 runs, then `BacktestRun.summary_metrics` contains `CAGR`, `max_drawdown`, `volatility`, `sharpe`, `hit_rate`, and `BacktestCompleted(status=success)` is emitted exactly once.
8. Given two finalized `BacktestRun`s with the same `BacktestWindow`, when `CompareTwoBacktests` runs, then the comparison object contains per-metric deltas; if both have the same `strategy_hash`, the comparison includes a `suspicious_identical_strategy` flag.
9. Given a backtest is in progress and the process is restarted, when a researcher queries the run, then it is marked `failed` (no auto-resume in v1) — backtests are batch artifacts, not long-running services.

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Cadence for backtest replay — daily? Weekly? Match production cadence configurable per strategy? | Researcher / domain expert | Open (lean: configurable; default daily for v1) |
| 2 | Slippage + commission model in v1 — zero (current proposal) or simple bps assumption (e.g., 5 bps each side)? | Domain expert | Open |
| 3 | Risk-free rate for Sharpe — fixed 0%? Configurable? Pulled from a yield-curve adapter? | Domain expert | Open |
| 4 | Scratch DB lifetime — drop on completion always, or keep for failed runs to allow inspection? | Researcher | Open (lean: keep for failed, drop for success) |
| 5 | LLM cost guardrail — should UC-1 estimate cost (universe size × frameworks × cadence ticks × tokens) and require researcher confirmation above a threshold? | Domain expert | Open (lean: yes, default $5 confirmation threshold) |
| 6 | Universe handling — fixed at config time or as-of `t`? Fixed risks survivorship bias; as-of requires a historical universe series. | Domain expert | Open (lean: as-of `t` to avoid survivorship bias) |
| 7 | Re-use of production read-models — backtests need an as-of variant of `FundamentalsView`, `InsiderClusterView`, etc. Are these new contracts on `market-data`, or duplicated in `backtesting`? | Architecture | Open (lean: extend `market-data` with as-of contracts; documented in `market-data-backtesting.md`) |
| 8 | Should UC-2 also invoke `rebalancing`/`execution` simulation in a future phase, with a fill model? Out of scope v1 but determines aggregate ownership for that future. | Domain expert | Open (Phase 2+) |

## Approval

| Reviewer | Role | Decision | Date |
|----------|------|----------|------|
| TBD | Domain expert | Pending | — |

## Related Docs

| Doc | Path |
|-----|------|
| Source PRD(s) | [../../../../input/prd.md](../../../../input/prd.md) |
| Context Map | [../../../architecture/context-map.md](../../../architecture/context-map.md) |
| Architecture | [../../../architecture/architecture.md](../../../architecture/architecture.md) |
| Ubiquitous Language | [../../../architecture/ubiquitous-language.md](../../../architecture/ubiquitous-language.md) |
| Event Catalog | [../../../architecture/event-catalog.md](../../../architecture/event-catalog.md) |
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0001-modular-monolith-hexagonal.md](../../../architecture/adr/0001-modular-monolith-hexagonal.md), [0005-llm-agents-return-strict-json.md](../../../architecture/adr/0005-llm-agents-return-strict-json.md)) |
