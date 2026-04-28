# Domain Event Catalog

**Version:** 0.1
**Last Updated:** 2026-04-28

## Event Summary

| Event | Publisher Context | Subscribers | Materialized Views | Criticality |
|-------|-----------------|-------------|-------------------|-------------|
| `PriceSnapshotIngested` | `market-data` | `monitoring`, `risk-regime`, `backtesting` | `LatestPriceView` | Medium |
| `FundamentalSnapshotIngested` | `market-data` | `alpha-generation`, `conviction-sizing` | `FundamentalsView` | Medium |
| `InsiderTransactionRecorded` | `market-data` | `catalyst-timing`, `alpha-generation` | `InsiderClusterView` | Medium |
| `ThirteenFFilingRecorded` | `market-data` | `alpha-generation` | `SmartMoneyHoldingsView` | Low |
| `AlphaCandidateGenerated` | `alpha-generation` | `conviction-sizing`, `catalyst-timing`, `portfolio-construction`, `backtesting` | `CandidateRankingView` | High |
| `ConvictionAssessed` | `conviction-sizing` | `portfolio-construction`, `backtesting` | `ConvictionView` | High |
| `CatalystIdentified` | `catalyst-timing` | `portfolio-construction`, `rebalancing`, `backtesting` | `CatalystView` | High |
| `RiskRegimeChanged` | `risk-regime` | `portfolio-construction`, `rebalancing` | `CurrentRegimeView` | High |
| `PortfolioTargetGenerated` | `portfolio-construction` | `rebalancing`, `backtesting` | `CurrentTargetView` | High |
| `RebalanceTriggered` | `rebalancing` | (internal — leads to `RebalancePlanProposed`) | — | Medium |
| `RebalancePlanProposed` | `rebalancing` | `execution` (via sync command), `monitoring` | — | High |
| `OrderSubmitted` | `execution` | `monitoring` | `OpenOrdersView` | High |
| `OrderFilled` | `execution` | `monitoring`, `portfolio-construction` (next pass) | `AccountPositionView` | High |
| `OrderRejected` | `execution` | `monitoring`, `rebalancing` | — | High |
| `PositionUpdated` | `execution` | `monitoring`, `portfolio-construction` (next pass) | `AccountPositionView` | High |
| `PortfolioHealthSnapshotTaken` | `monitoring` | (consumed via read-model only) | `PortfolioHealthView` | Low |
| `RiskAlertRaised` | `monitoring` | `risk-regime`, `rebalancing` | `OpenAlertsView` | High |
| `BacktestCompleted` | `backtesting` | (researcher-facing only) | `BacktestResultsView` | Low |
| `LoopOverrun` (operational) | `orchestrator` | `monitoring` | — | Medium |

All events carry a `run_id` for correlation across one Trading-Loop pass.

## Event Definitions

### `PriceSnapshotIngested`

- **Publisher:** `signal-intelligence/market-data`
- **Trigger:** A daily (or intra-day) price snapshot is persisted for one ticker.
- **Criticality:** Medium — downstream contexts can recover from a missed event by re-querying the read-model on the next pass.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | Unique event id (idempotency key). |
| `run_id` | string | Yes | Trading-Loop run correlation id. |
| `ticker` | string | Yes | Ticker symbol. |
| `as_of` | datetime (UTC) | Yes | Snapshot timestamp. |
| `open` | decimal | Yes | OHLCV — open. |
| `high` | decimal | Yes | OHLCV — high. |
| `low` | decimal | Yes | OHLCV — low. |
| `close` | decimal | Yes | OHLCV — close. |
| `volume` | int | Yes | OHLCV — volume. |
| `currency` | string | Yes | ISO-4217. v1: always `USD`. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `monitoring` | Update last-known price for held positions. | `LatestPriceView` | Retry x3, then dead-letter. |
| `risk-regime` | Update breadth/volatility series. | `LatestPriceView` | Retry x3, then skip (next pass recovers). |
| `backtesting` | Append to historical series (when running). | (per-backtest scratch DB) | Fail backtest run on dead-letter. |

**Integration Contracts:** `market-data-monitoring.md`, `market-data-risk-regime.md`, `market-data-backtesting.md`.

---

### `FundamentalSnapshotIngested`

- **Publisher:** `signal-intelligence/market-data`
- **Trigger:** Updated financial-statement values are persisted for one ticker.
- **Criticality:** Medium.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | Idempotency key. |
| `run_id` | string | Yes | |
| `ticker` | string | Yes | |
| `as_of` | date | Yes | Period end date. |
| `period` | string | Yes | `quarterly` or `annual`. |
| `roic` | decimal | No | Return on invested capital. |
| `earnings_yield` | decimal | No | E/P. |
| `revenue_growth_yoy` | decimal | No | Year-over-year revenue growth. |
| `gross_margin_trend` | decimal | No | Δ gross margin vs. prior period. |
| `analyst_coverage` | int | No | Number of analyst firms covering. |
| `market_cap` | decimal | No | Current market cap (USD). |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `alpha-generation` | Re-evaluate eligibility filters; mark candidate stale if values change. | `FundamentalsView` | Retry x3, then dead-letter. |
| `conviction-sizing` | Recompute moat/reinvestment scores on next pass. | `FundamentalsView` | Retry x3. |

**Integration Contracts:** `market-data-alpha-generation.md`, `market-data-conviction-sizing.md`.

---

### `InsiderTransactionRecorded`

- **Publisher:** `signal-intelligence/market-data`
- **Trigger:** A Form 4 filing has been parsed from SEC EDGAR.
- **Criticality:** Medium.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `ticker` | string | Yes | |
| `insider_name` | string | Yes | |
| `insider_role` | string | Yes | `officer` / `director` / `10pct_holder`. |
| `transaction_type` | string | Yes | `buy` / `sell`. |
| `shares` | int | Yes | |
| `price` | decimal | Yes | |
| `transaction_date` | date | Yes | |
| `filing_date` | date | Yes | |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `catalyst-timing` | Recompute insider-cluster signal on the ticker's open candidates. | `InsiderClusterView` | Retry x3, then dead-letter. |
| `alpha-generation` | Feeds the Insider+13F framework score. | `InsiderClusterView` | Retry x3. |

**Integration Contracts:** `market-data-catalyst-timing.md`, `market-data-alpha-generation.md`.

---

### `ThirteenFFilingRecorded`

- **Publisher:** `signal-intelligence/market-data`
- **Trigger:** A 13F-HR filing has been parsed from SEC EDGAR.
- **Criticality:** Low — quarterly cadence; rerun-friendly.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `filer_cik` | string | Yes | SEC filer Central Index Key. |
| `filer_name` | string | Yes | |
| `period_end` | date | Yes | Quarter-end date. |
| `filing_date` | date | Yes | |
| `holdings` | array<{ticker, shares, market_value}> | Yes | List of disclosed positions. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `alpha-generation` | Update smart-money signal for holdings of tracked filers. | `SmartMoneyHoldingsView` | Retry x3. |

**Integration Contracts:** `market-data-alpha-generation.md`.

---

### `AlphaCandidateGenerated`

- **Publisher:** `signal-intelligence/alpha-generation`
- **Trigger:** A `ScreeningRun` produces a new (or re-scored) `AlphaCandidate`.
- **Criticality:** High — drives the rest of the pipeline.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `screening_run_id` | UUID | Yes | Parent `ScreeningRun` id. |
| `ticker` | string | Yes | |
| `alpha_score` | int (0–100) | Yes | Composite. |
| `framework_scores` | map<framework_name, int> | Yes | Per-framework sub-scores. |
| `reason` | string | Yes | Short narrative (LLM-produced, ≤ 280 chars). |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `conviction-sizing` | Trigger a `ConvictionAssessment` for this ticker. | `CandidateRankingView` | Retry x5, then dead-letter (alert raised). |
| `catalyst-timing` | Trigger a `CatalystAssessment` for this ticker. | `CandidateRankingView` | Retry x5, then dead-letter. |
| `portfolio-construction` | Buffer for the current pass's composition step. | `CandidateRankingView` | Retry x3. |
| `backtesting` | Append to backtest scratch state. | (per-backtest) | Fail backtest run. |

**Integration Contracts:** `alpha-generation-conviction-sizing.md`, `alpha-generation-catalyst-timing.md`, `alpha-generation-portfolio-construction.md`.

---

### `ConvictionAssessed`

- **Publisher:** `signal-intelligence/conviction-sizing`
- **Trigger:** `ConvictionService.assess(candidate)` completes.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `ticker` | string | Yes | |
| `conviction_grade` | enum `A`/`B`/`C`/`D` | Yes | |
| `target_weight_min` | decimal | Yes | Lower bound of `TargetWeightRange`. |
| `target_weight_max` | decimal | Yes | Upper bound. |
| `reason` | string | Yes | Short narrative (≤ 280 chars). |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `portfolio-construction` | Use as sizing input for the candidate. | `ConvictionView` | Retry x5, then dead-letter. |

**Integration Contracts:** `conviction-portfolio-construction.md`.

---

### `CatalystIdentified`

- **Publisher:** `signal-intelligence/catalyst-timing`
- **Trigger:** `CatalystService.evaluate(candidate)` returns a non-pass `EntryAction`.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `ticker` | string | Yes | |
| `catalyst_type` | enum (`EarningsInflection`, `Spinoff`, `Buyback`, `InsiderClusterBuy`, `RegulatoryChange`, `MarginRecovery`) | Yes | |
| `catalyst_score` | int (0–100) | Yes | |
| `entry_action` | enum (`buy_now`, `wait`) | Yes | `pass` outcomes do not raise this event. |
| `timeframe_months` | string | Yes | e.g., `"3-9"`. |
| `reason` | string | Yes | Short narrative (≤ 280 chars). |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `portfolio-construction` | Bias the timing component of `target_weight`. | `CatalystView` | Retry x5. |
| `rebalancing` | Hold a fresh catalyst on a held name as input to drift evaluation. | `CatalystView` | Retry x3. |

**Integration Contracts:** `catalyst-portfolio-construction.md`.

---

### `RiskRegimeChanged`

- **Publisher:** `portfolio-management/risk-regime`
- **Trigger:** `RiskRegimeService.recompute()` produces a regime different from the previous state.
- **Criticality:** High — changes exposure caps for the entire portfolio.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `previous_regime` | enum | Yes | `risk_on`/`neutral`/`risk_off`/`capital_preservation`. |
| `current_regime` | enum | Yes | New regime. |
| `max_equity_exposure` | decimal | Yes | E.g., 0.75. |
| `cash_target` | decimal | Yes | E.g., 0.15. |
| `max_position` | decimal | Yes | E.g., 0.08. |
| `reason` | string | Yes | Short narrative (≤ 280 chars). |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `portfolio-construction` | Apply new caps in next composition. | `CurrentRegimeView` | Retry x5; if dead-lettered, orchestrator halts loop and raises `RiskAlertRaised`. |
| `rebalancing` | Trigger immediate rebalance evaluation. | `CurrentRegimeView` | Retry x5. |

**Integration Contracts:** `risk-regime-portfolio-construction.md`, `risk-regime-rebalancing.md`.

---

### `PortfolioTargetGenerated`

- **Publisher:** `portfolio-management/portfolio-construction`
- **Trigger:** A new `PortfolioTarget` is persisted at the end of a composition step.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `target_id` | UUID | Yes | `PortfolioTarget` aggregate id. |
| `as_of` | datetime | Yes | |
| `regime` | enum | Yes | Regime applied. |
| `positions` | array<{ticker, target_weight, action}> | Yes | One per `TargetPosition`. |
| `cash_target` | decimal | Yes | |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `rebalancing` | Compare against current positions; emit `RebalanceTriggered` if drift exceeds threshold. | `CurrentTargetView` | Retry x5; halt loop on dead-letter. |
| `backtesting` | Record the target for the backtest window. | (per-backtest) | Fail backtest run. |

**Integration Contracts:** `portfolio-construction-rebalancing.md`.

---

### `RebalanceTriggered`

- **Publisher:** `portfolio-management/rebalancing`
- **Trigger:** A `RebalanceTrigger` (drift / regime / catalyst / earnings / insider-sell) fires.
- **Criticality:** Medium — internal to the rebalancing context; precedes `RebalancePlanProposed`.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `trigger_type` | enum | Yes | `WeightDriftExceeded`, `ConvictionChanged`, `RiskRegimeChanged`, `PriceMoveExceeded`, `EarningsEvent`, `InsiderSelling`. |
| `tickers` | array<string> | Yes | Affected tickers (may be portfolio-wide). |
| `reason` | string | Yes | |

**Subscribers & Projections:** Internal — used to start `RebalancePlan` composition.

**Integration Contracts:** —

---

### `RebalancePlanProposed`

- **Publisher:** `portfolio-management/rebalancing`
- **Trigger:** A `RebalancePlan` is persisted and ready for submission.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `plan_id` | UUID | Yes | |
| `target_id` | UUID | Yes | Source `PortfolioTarget`. |
| `actions` | array<{ticker, side, qty, order_type, limit_price?}> | Yes | One per `RebalanceAction`. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `execution` | Receives via the sync `SubmitRebalancePlan` command, not via async subscription. | — | Sync — failures bubble up. |
| `monitoring` | Records plan for audit. | — | Retry x3. |

**Integration Contracts:** `rebalancing-execution.md`.

---

### `OrderSubmitted`

- **Publisher:** `trade-operations/execution`
- **Trigger:** IBKR has acknowledged receipt of an order.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `order_id` | UUID | Yes | Internal order id. |
| `ibkr_order_id` | string | Yes | IBKR order id. |
| `plan_id` | UUID | Yes | Source plan. |
| `ticker` | string | Yes | |
| `side` | enum (`BUY`/`SELL`) | Yes | |
| `qty` | int | Yes | |
| `order_type` | enum | Yes | |
| `limit_price` | decimal | No | Required when `order_type=LIMIT`. |
| `paper_trading` | bool | Yes | Per ADR-0004. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `monitoring` | Track open orders for staleness/fill latency. | `OpenOrdersView` | Retry x3. |

**Integration Contracts:** `execution-monitoring.md`.

---

### `OrderFilled`

- **Publisher:** `trade-operations/execution`
- **Trigger:** IBKR reports a partial or complete fill.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `order_id` | UUID | Yes | |
| `fill_id` | UUID | Yes | |
| `ticker` | string | Yes | |
| `qty` | int | Yes | Filled this fill (not cumulative). |
| `price` | decimal | Yes | |
| `commission` | decimal | Yes | |
| `filled_at` | datetime | Yes | |
| `is_complete` | bool | Yes | True if order fully filled. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `monitoring` | Update P&L and fill latency metrics. | `AccountPositionView` | Retry x5; on dead-letter raise `RiskAlertRaised`. |
| `portfolio-construction` (next pass) | Reads via `AccountPositionView`. | `AccountPositionView` | — |

**Integration Contracts:** `execution-monitoring.md`.

---

### `OrderRejected`

- **Publisher:** `trade-operations/execution`
- **Trigger:** IBKR rejects an order, or an internal pre-trade validation fails.
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `order_id` | UUID | Yes | |
| `plan_id` | UUID | Yes | |
| `ticker` | string | Yes | |
| `reason_code` | string | Yes | Source code (IBKR or internal). |
| `reason` | string | Yes | Human-readable reason. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `monitoring` | Raise `RiskAlertRaised` on repeated rejections. | — | Retry x3. |
| `rebalancing` | Mark the action failed; consider re-plan on next pass. | — | Retry x3. |

**Integration Contracts:** `execution-monitoring.md`, `rebalancing-execution.md`.

---

### `PositionUpdated`

- **Publisher:** `trade-operations/execution`
- **Trigger:** `AccountPosition` aggregate state changes (after fills or reconciliation).
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `ticker` | string | Yes | |
| `quantity` | int | Yes | New total quantity (signed: + long, - short). |
| `avg_cost` | decimal | Yes | New cost basis. |
| `market_value` | decimal | Yes | At time of update. |
| `as_of` | datetime | Yes | |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `monitoring` | Recompute exposures and drawdown. | `AccountPositionView` | Retry x5. |
| `portfolio-construction` (next pass) | Reads via `AccountPositionView`. | `AccountPositionView` | — |

**Integration Contracts:** `execution-monitoring.md`.

---

### `PortfolioHealthSnapshotTaken`

- **Publisher:** `trade-operations/monitoring`
- **Trigger:** `MonitoringService.snapshot()` runs (every loop pass + on demand).
- **Criticality:** Low — view is the source of truth; the event is for archival.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `as_of` | datetime | Yes | |
| `total_value` | decimal | Yes | |
| `cash` | decimal | Yes | |
| `drawdown` | decimal | Yes | |
| `volatility_30d` | decimal | Yes | |
| `sector_exposure` | map<string, decimal> | Yes | |
| `benchmark_return_30d` | decimal | No | |

**Subscribers & Projections:** None directly — read-model only.

**Integration Contracts:** —

---

### `RiskAlertRaised`

- **Publisher:** `trade-operations/monitoring`
- **Trigger:** A health metric crosses a threshold (sector concentration, drawdown breach, repeated order rejections, stale price data).
- **Criticality:** High.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `alert_id` | UUID | Yes | |
| `severity` | enum (`info`/`warning`/`critical`) | Yes | |
| `category` | string | Yes | E.g., `sector_concentration`, `drawdown_breach`. |
| `description` | string | Yes | |
| `recommended_action` | string | No | E.g., "reduce exposure". |
| `affected_tickers` | array<string> | No | |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `risk-regime` | Consider regime downgrade. | `OpenAlertsView` | Retry x5; halt loop on dead-letter for `critical` severity. |
| `rebalancing` | Trigger plan re-evaluation. | `OpenAlertsView` | Retry x5. |

**Integration Contracts:** `monitoring-risk-regime.md`, `monitoring-rebalancing.md`.

---

### `BacktestCompleted`

- **Publisher:** `research-validation/backtesting`
- **Trigger:** A `BacktestRun` finishes successfully (or fails).
- **Criticality:** Low — researcher-facing.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `backtest_id` | UUID | Yes | |
| `strategy_hash` | string | Yes | Hash of strategy config. |
| `window_start` | date | Yes | |
| `window_end` | date | Yes | |
| `status` | enum (`success`/`failed`) | Yes | |
| `summary_metrics` | object | No | CAGR, max drawdown, Sharpe, etc. |

**Subscribers & Projections:** Researcher-facing report only.

**Integration Contracts:** —

---

### `LoopOverrun` (operational)

- **Publisher:** `orchestrator` (cross-cutting, not a bounded context)
- **Trigger:** A Trading-Loop pass exceeds 80% of the configured cadence interval.
- **Criticality:** Medium.

**Payload:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | Yes | |
| `run_id` | string | Yes | |
| `cadence_seconds` | int | Yes | Configured interval. |
| `elapsed_seconds` | int | Yes | Actual elapsed. |
| `slowest_step` | string | Yes | Context whose step exceeded its budget. |

**Subscribers & Projections:**

| Context | Reaction | Projected Into View | Failure Handling |
|---------|----------|-------------------|-----------------|
| `monitoring` | May raise a `RiskAlertRaised(severity=warning)` after N overruns. | — | Retry x3. |

**Integration Contracts:** —

---

## Event Flow Diagrams

```text
─── Per Trading-Loop pass (run_id = R) ───

market-data  ─PriceSnapshotIngested─►        monitoring (LatestPriceView)
             ─FundamentalSnapshotIngested─►  alpha-generation, conviction-sizing (FundamentalsView)
             ─InsiderTransactionRecorded─►   catalyst-timing, alpha-generation
             ─ThirteenFFilingRecorded─►      alpha-generation

alpha-generation ─AlphaCandidateGenerated─►  conviction-sizing
                                          ─► catalyst-timing
                                          ─► portfolio-construction

conviction-sizing ─ConvictionAssessed─►      portfolio-construction
catalyst-timing   ─CatalystIdentified─►      portfolio-construction (and rebalancing for held names)

risk-regime ─RiskRegimeChanged─►             portfolio-construction, rebalancing

portfolio-construction ─PortfolioTargetGenerated─► rebalancing

rebalancing ─RebalanceTriggered─► (internal) ─► RebalancePlanProposed ─SYNC SubmitRebalancePlan─► execution

execution   ─OrderSubmitted─►   monitoring
            ─OrderFilled─►      monitoring  (AccountPositionView)
            ─OrderRejected─►    monitoring, rebalancing
            ─PositionUpdated─►  monitoring  (AccountPositionView)

monitoring  ─PortfolioHealthSnapshotTaken─► (PortfolioHealthView only)
            ─RiskAlertRaised─► risk-regime, rebalancing   (feedback loop closes here)

orchestrator ─LoopOverrun─►  monitoring
```

## Event Conventions

| Convention | Rule |
|-----------|------|
| **Naming** | Past tense verb; one-event-per-state-change (e.g., `OrderFilled`, not `OrderFillEvent`). Commands, by contrast, are imperative (`SubmitRebalancePlan`). |
| **Payload format** | Pydantic dataclasses serialized to JSON on the bus; UTC ISO-8601 datetimes; decimals as strings to preserve precision. |
| **Delivery guarantee** | At-least-once (in-process bus may duplicate on restart). |
| **Ordering** | Per-aggregate FIFO only; no cross-aggregate ordering guarantees. Subscribers must be commutative across aggregates. |
| **Idempotency** | All subscribers must dedupe on `(event_name, event_id)` — this is enforced by the `EventBus` wrapper. |
| **Schema ownership** | Defined in this catalog; per-context `events.asyncapi.yaml` mirrors these definitions. Markdown integration contracts reference event names — do not redefine payloads. |
| **Versioning** | HR-7: append-only. Breaking changes require a new event name and a deprecation window of at least one full release cycle. |
| **Correlation** | Every event carries `run_id` for one Trading-Loop pass; queries can reconstruct the per-pass causal chain. |

## Dead Letter / Error Events

| Error Event | Source | Condition | Handler |
|------------|--------|-----------|---------|
| `EventDeadLettered` | `EventBus` (cross-cutting) | A subscriber exceeded its retry budget for any event. | Logged; `monitoring` raises `RiskAlertRaised(severity=warning)` if the dropped event was High criticality, `critical` for execution-side events. |
| `OrderRejected` | `execution` | (Domain event, not infra error — see above.) | See subscribers above. |
| `AgentParseError` (operational log, not a bus event) | Any context using LLM agents | LLM returned non-parseable JSON after retries (ADR-0005). | Logged with prompt + response hash; the failing pipeline step is skipped and `monitoring` raises a `warning` alert. |
| `IbkrConnectionLost` (operational log) | `execution` adapter | IBKR session disconnect. | Reconnect with back-off; `monitoring` raises `critical` alert if not recovered within 60 s. |

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md](context-map.md) |
| Architecture | [architecture.md](architecture.md) |
| Ubiquitous Language | [ubiquitous-language.md](ubiquitous-language.md) |
| ADR Index | [adr/](adr/) |
| Per-context AsyncAPI specs | `domains/<ctx>/events.asyncapi.yaml` (created in later sessions) |
| Per-context bounded-context docs | `domains/<ctx>/bounded-context.md` (created in later sessions) |
