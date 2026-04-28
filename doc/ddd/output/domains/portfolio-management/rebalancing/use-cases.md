# Intent Translation: portfolio-management/rebalancing

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.6 Rebalancing Agent; §5 Execution Loop), `doc/ddd/input/investment-frameworks.md` (frameworks 8 Marks, 10 Burry, 7 Pabrai)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Detect when the live portfolio (`AccountPositionView`) has diverged from the latest `PortfolioTarget` enough to warrant action — either by `WeightDriftExceeded`, an upstream signal change (`RiskRegimeChanged`, `CatalystIdentified`, `ConvictionChanged`), an external event (`EarningsEvent`, `PriceMoveExceeded`), or `InsiderSelling` on a held name. When triggered, compose a `RebalancePlan` (a list of `RebalanceAction`s with `side`, `qty`, `order_type`, `limit_price?`) and submit it **synchronously** to `execution` so broker rejections fail fast (ADR-0003). Internally raises `RebalanceTriggered`; externally emits `RebalancePlanProposed` for the audit / monitoring trail.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `portfolio-management/rebalancing` | Yes | Owns `RebalancePlan` aggregate. |
| `portfolio-management/portfolio-construction` | No | Provides `PortfolioTargetGenerated` + `CurrentTargetView`. |
| `portfolio-management/risk-regime` | No | `RiskRegimeChanged` is a trigger. |
| `signal-intelligence/catalyst-timing` | No | `CatalystIdentified` (esp. `EarningsInflection`) is a trigger. |
| `signal-intelligence/conviction-sizing` | No | A `ConvictionAssessed` whose grade changes for a held name is a trigger (`ConvictionChanged`). |
| `signal-intelligence/market-data` | No | Provides `LatestPriceView` for `WeightDriftExceeded` and `PriceMoveExceeded` computation; emits `InsiderTransactionRecorded` (`InsiderSelling`). |
| `trade-operations/execution` | No | Receives `SubmitRebalancePlan` (sync); emits `OrderRejected` back. |
| `trade-operations/monitoring` | No | `RiskAlertRaised` is a trigger; consumes `RebalancePlanProposed` for audit. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Rebalancing Agent (PRD §4.6) | RebalanceService | `RebalanceService` | Existing |
| Triggers (PRD §4.6 list) | RebalanceTrigger (enum) | `RebalanceTrigger ∈ {WeightDriftExceeded, ConvictionChanged, RiskRegimeChanged, PriceMoveExceeded, EarningsEvent, InsiderSelling}` | Existing |
| `weight_drift > 25%` (PRD §4.6) | WeightDrift threshold | `WeightDrift = |current_weight − target_weight| / target_weight` | Existing |
| `price_move > 20%` (PRD §4.6) | PriceMoveExceeded threshold | `Δ% over lookback window` | Existing |
| Rebalance plan (PRD §4.6 output) | RebalancePlan + RebalanceAction | `RebalancePlan`, `RebalanceAction` | Existing |
| `action: trim` (PRD §4.6 example) | RebalanceAction.side / RebalanceAction direction | `side ∈ {BUY, SELL}` derived from current vs. target weight | Existing |

## Use Cases

### UC-1: EvaluateDriftOnNewTarget

- **Actor:** `rebalancing` event subscriber.
- **Trigger:** `PortfolioTargetGenerated` from `portfolio-construction`.
- **Goal:** Compare each `TargetPosition` against the corresponding `AccountPosition`; if any `WeightDrift` exceeds the configured threshold (default 25% per PRD §4.6) or `target.action ∈ {open, close}` regardless of drift, raise `RebalanceTriggered(trigger_type=WeightDriftExceeded, tickers=[…])` and proceed to UC-7 (compose plan).
- **Primary Aggregate:** `RebalancePlan` (composed downstream of the trigger).
- **Pre-conditions:** `CurrentTargetView` populated for the new `target_id`; `AccountPositionView` available; `LatestPriceView` available for current weight calculation.
- **Outcome (success):** `RebalanceTriggered` raised internally; `RebalancePlan` composed and `RebalancePlanProposed` emitted; sync `SubmitRebalancePlan` issued to `execution`.
- **Outcome (failure):** No drift exceeds threshold and no `open`/`close` actions → no-op (no event). Stale positions (> 60s) → defer to next pass; log warning.
- **Invariants Touched:** Only one `RebalancePlan` per `(target_id, trigger_type)` in flight; `WeightDrift` calculated on absolute weights (using current `LatestPriceView` for valuation), not on shares.
- **Cross-context Calls:** Reads `CurrentTargetView`, `AccountPositionView`, `LatestPriceView`; emits `RebalanceTriggered` (internal), `RebalancePlanProposed`; sync command `SubmitRebalancePlan` to `execution`.

### UC-2: TriggerOnRegimeChange

- **Actor:** `rebalancing` event subscriber.
- **Trigger:** `RiskRegimeChanged` from `risk-regime`.
- **Goal:** Force a fresh evaluation against the **current** `PortfolioTarget` (which may not yet reflect the new regime — `portfolio-construction` UC-2 will recompose, but `rebalancing` may need to act first if the regime is `capital_preservation`). Emit `RebalanceTriggered(trigger_type=RiskRegimeChanged)`; compose plan if any action required.
- **Primary Aggregate:** `RebalancePlan`.
- **Pre-conditions:** Current `PortfolioTarget` exists.
- **Outcome (success):** Plan composed using the **post-recompose** target if `portfolio-construction` UC-2 has already published it (preferred — wait up to a configurable timeout, default 5s); else use the pre-recompose target with a `WARN_REGIME_PRE_RECOMPOSE` flag.
- **Outcome (failure):** No target exists yet → no-op (next `PortfolioTargetGenerated` will run UC-1). Sync command to `execution` rejected (e.g., paper-trading off, IBKR down) → propagate failure; orchestrator decides whether to halt loop.
- **Invariants Touched:** Regime-driven plans must respect the **new** `RiskRegimeState.max_position` and `max_equity_exposure` even if the target lags; if the pre-recompose target violates the new caps, the plan must trim first (defensive).
- **Cross-context Calls:** Subscribes to `RiskRegimeChanged`; reads `CurrentTargetView`, `CurrentRegimeView`, `AccountPositionView`; emits `RebalanceTriggered`, `RebalancePlanProposed`; sync `SubmitRebalancePlan`.

### UC-3: TriggerOnCatalystOrConvictionChange

- **Actor:** `rebalancing` event subscriber.
- **Trigger:** `CatalystIdentified` (especially `catalyst_type ∈ {EarningsInflection, InsiderClusterBuy}`) **for a held ticker**, OR `ConvictionAssessed` whose `conviction_grade` differs from the previous assessment for a held ticker (`ConvictionChanged`).
- **Goal:** Emit `RebalanceTriggered(trigger_type ∈ {EarningsEvent, ConvictionChanged})`; consult current target — if no plan change is warranted (e.g., catalyst is already priced into target), no-op; else compose plan.
- **Primary Aggregate:** `RebalancePlan`.
- **Pre-conditions:** Ticker is in `AccountPositionView` (held) — events for non-held tickers are ignored at this stage (those flow through `portfolio-construction` instead).
- **Outcome (success):** Plan composed iff target weight (per latest target) differs materially from current weight given the new signal; else no-op.
- **Outcome (failure):** Same as UC-1.
- **Invariants Touched:** This BC does not re-derive target weights — it consumes the latest `PortfolioTarget`. If the catalyst implies a new weight, that recomposition belongs to `portfolio-construction` (separation per PRD §2 design principle).
- **Cross-context Calls:** Subscribes to `CatalystIdentified`, `ConvictionAssessed`; reads `CurrentTargetView`, `AccountPositionView`; emits `RebalanceTriggered`, `RebalancePlanProposed`; sync `SubmitRebalancePlan`.

### UC-4: TriggerOnPriceMove

- **Actor:** Trading Loop orchestrator (cadence-driven monitor).
- **Trigger:** Cadence tick — for each held position, compute price move over the configured lookback (default 1 trading day); if `|Δ%| > 20%` (PRD §4.6) raise `RebalanceTriggered(trigger_type=PriceMoveExceeded)`.
- **Primary Aggregate:** `RebalancePlan`.
- **Pre-conditions:** `LatestPriceView` populated; held positions known.
- **Outcome (success):** Plan composed if the price move pushes `WeightDrift` over its threshold or breaks `RiskRegimeState.max_position`.
- **Outcome (failure):** Stale price → skip ticker; log warning. (`monitoring` will raise `RiskAlertRaised(category=stale_price)`.)
- **Invariants Touched:** Price-move trigger does not bypass drift logic — it is a fast path for catching intra-day spikes between cadence ticks; the resulting plan still respects all regime + target constraints.
- **Cross-context Calls:** Reads `LatestPriceView`, `AccountPositionView`, `CurrentTargetView`; emits `RebalanceTriggered`, `RebalancePlanProposed`; sync `SubmitRebalancePlan`.

### UC-5: TriggerOnInsiderSelling

- **Actor:** `rebalancing` event subscriber.
- **Trigger:** `InsiderTransactionRecorded` from `market-data` with `transaction_type=sell` for a held ticker, where the seller is an officer/director and `shares` exceed a configured significance threshold (TBD with domain expert).
- **Goal:** Emit `RebalanceTriggered(trigger_type=InsiderSelling)`; consult target and `catalyst-timing` view — if the sell signal contradicts a `buy_now` catalyst with strong score, downgrade to a `trim` (not `close`); else propose `close` or `trim` per current target.
- **Primary Aggregate:** `RebalancePlan`.
- **Pre-conditions:** Held ticker; seller is Section 16 insider (officer/director/10pct-holder per `market-data` ubiquitous-language).
- **Outcome (success):** Plan composed; for material insider sells, default action is `trim` to half current weight (TBD with domain expert).
- **Outcome (failure):** Below significance threshold → no-op. 10%-holder routine sales (e.g., scheduled 10b5-1) — see open question.
- **Invariants Touched:** Insider-sell-driven actions must not exceed `max_single_position` cap on remaining holdings; never increase exposure on an insider-sell trigger.
- **Cross-context Calls:** Subscribes to `InsiderTransactionRecorded`; reads `AccountPositionView`, `CurrentTargetView`, `CatalystView`; emits `RebalanceTriggered`, `RebalancePlanProposed`; sync `SubmitRebalancePlan`.

### UC-6: TriggerOnRiskAlert

- **Actor:** `rebalancing` event subscriber.
- **Trigger:** `RiskAlertRaised` from `monitoring` with `severity ∈ {warning, critical}`.
- **Goal:** Map the alert `category` (`sector_concentration`, `drawdown_breach`, `stale_price`, repeated rejections) to a defensive action — typically trim concentrated sectors or move to higher cash; emit `RebalanceTriggered(trigger_type=…)`. (Note: the catalog enum does not include a dedicated `RiskAlert` trigger type; reuse the closest applicable type and record the originating `alert_id` in `reason`.)
- **Primary Aggregate:** `RebalancePlan`.
- **Pre-conditions:** Alert is one of the actionable categories (mapping table TBD with domain expert).
- **Outcome (success):** Plan composed and submitted; defensive only (never increase exposure on a risk alert).
- **Outcome (failure):** Alert category not actionable → acknowledged (no plan); recorded in audit.
- **Invariants Touched:** Defensive-only — analogous to `risk-regime` UC-2 monotonic-downgrade rule but at the position/sector level.
- **Cross-context Calls:** Subscribes to `RiskAlertRaised`; reads `AccountPositionView`, `CurrentTargetView`; emits `RebalanceTriggered`, `RebalancePlanProposed`; sync `SubmitRebalancePlan`.

### UC-7: ComposeAndSubmitRebalancePlan

- **Actor:** Internal — invoked by UC-1…UC-6 once a trigger fires.
- **Trigger:** `RebalanceTriggered` (internal event).
- **Goal:** Translate the trigger + current target + current positions into a concrete `RebalancePlan` with one `RebalanceAction` per affected ticker (`side`, `qty`, `order_type=LIMIT` per PRD §4.7 default, optional `limit_price`); persist it; emit `RebalancePlanProposed`; issue sync `SubmitRebalancePlan(plan_id)` to `execution`.
- **Primary Aggregate:** `RebalancePlan`.
- **Pre-conditions:** Internal trigger raised; target + positions + prices available.
- **Outcome (success):** Plan persisted; event emitted; sync command returns success → done. Sync command returns `OrderRejected` → mark plan partially failed; raise `RebalanceTriggered` again on next pass for the failed actions (bounded retry per ADR-0003 — exact count TBD with domain expert).
- **Outcome (failure):** No actionable change after recomputing (e.g., trigger fired but drift back within band by the time plan composes) → drop plan, log `RebalancePlanEmpty` (no event). Sync `SubmitRebalancePlan` raises a transport error (IBKR down) → propagate to orchestrator; loop may halt or retry per `monitoring`.
- **Invariants Touched:** Per `RebalanceAction`: `side` derived from `(current_weight − target_weight)`; `qty` = round-down to share lot of `(|Δ weight| × portfolio_value / price)`; `order_type=LIMIT` by default (PRD §4.7); `max_order_size ≤ 0.05 × portfolio_value` per action (PRD §4.7) — split into multiple actions if a single ticker's required move exceeds the cap. One `RebalancePlan` per `(target_id, trigger_type, run_id)`.
- **Cross-context Calls:** Sync command `SubmitRebalancePlan(plan_id)` to `execution`; emits `RebalancePlanProposed`.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `RebalanceTrigger ∈ {WeightDriftExceeded, ConvictionChanged, RiskRegimeChanged, PriceMoveExceeded, EarningsEvent, InsiderSelling}` | `RebalancePlan` | New (anchor: ubiquitous-language) |
| `WeightDrift = |current_weight − target_weight| / target_weight`; default threshold `0.25` | `RebalanceService` | New (anchor: PRD §4.6) |
| `PriceMoveExceeded` threshold default `0.20` over 1-day lookback | `RebalanceService` | New (anchor: PRD §4.6) |
| One `RebalancePlan` per `(target_id, trigger_type, run_id)` | `RebalancePlan` | New |
| `RebalanceAction.side` derived from `(current_weight − target_weight)` | `RebalanceAction` | New |
| Per-action max size ≤ `0.05 × portfolio_value` (PRD §4.7); split if exceeded | `RebalanceAction` | New |
| `order_type=LIMIT` by default (PRD §4.7) | `RebalanceAction` | New |
| Defensive-only triggers (UC-2 regime to lower, UC-5 insider sell, UC-6 risk alert) never increase exposure | `RebalanceService` | New |
| `SubmitRebalancePlan` is synchronous (ADR-0003) — broker rejection bubbles up | `RebalanceService` | New (anchor: ADR-0003) |
| Held tickers absent from current target appear as `close` actions in plan | `RebalancePlan` | New (mirrors `portfolio-construction` invariant) |
| Plans composing zero actions are dropped, no `RebalancePlanProposed` emitted | `RebalanceService` | New (noise suppression) |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `portfolio-construction` | Event: `PortfolioTargetGenerated` + read-model `CurrentTargetView` | New target → drift check. |
| Inbound | `risk-regime` | Event: `RiskRegimeChanged` + read-model `CurrentRegimeView` | Regime trigger + cap enforcement. |
| Inbound | `catalyst-timing` | Event: `CatalystIdentified` | Earnings / catalyst trigger on held names. |
| Inbound | `conviction-sizing` | Event: `ConvictionAssessed` | `ConvictionChanged` trigger. |
| Inbound | `market-data` | Event: `InsiderTransactionRecorded` (sell side); Read-model `LatestPriceView` | `InsiderSelling`, `PriceMoveExceeded`, weight valuation. |
| Inbound | `monitoring` | Event: `RiskAlertRaised` | Defensive trigger. |
| Inbound | `execution` | Event: `OrderRejected` + sync command return | Plan retry / failure handling. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `RiskTolerance`, `MaxDrawdown` shape thresholds. |
| Outbound | `execution` | Sync command: `SubmitRebalancePlan(plan_id)` | Plan submission (per ADR-0003). |
| Outbound | `monitoring` | Event: `RebalancePlanProposed` | Audit / fill-latency tracking. |
| Outbound | (internal) | Event: `RebalanceTriggered` | Trigger → composition handoff inside the BC. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Trigger detectors (drift, price-move, regime, catalyst, conviction, insider-sell, risk-alert mappings) | `rebalancing` | TBD |
| Plan composer (`RebalancePlan` + `RebalanceAction` derivation) | `rebalancing` | TBD |
| Limit-price policy (mid? last? last + N bps?) | `rebalancing` | TBD |
| Sync-submit error policy (retry, halt, escalate) | `rebalancing` × `orchestrator` | TBD |

## Constraints & Prohibited Actions

1. **No direct IBKR calls** — plan submission flows through `execution` only (HR-1 + ADR-0003).
2. **No re-derivation of target weights** — `rebalancing` consumes `PortfolioTarget`; weight changes belong to `portfolio-construction`. (PRD §2 separation principle.)
3. **Sync command for `SubmitRebalancePlan`** (ADR-0003) — broker rejection must fail fast so the rebalancer can decide whether to retry or escalate; do not fire-and-forget.
4. **Defensive-only on risk-driven triggers** — UC-2 (regime tighter), UC-5 (insider sell), UC-6 (risk alert) may trim or close, never open or increase.
5. **Per-action size cap** — `qty × price ≤ 0.05 × portfolio_value` (PRD §4.7); split larger moves into N actions; the plan still must stay coherent (no two actions on the same ticker in the same plan; later passes handle the next slice).
6. **`order_type=LIMIT` default** (PRD §4.7); other order types require explicit policy override per ticker (TBD).
7. **Idempotency** — duplicate inbound events (`(event_name, event_id)`) result in at most one trigger evaluation; duplicate `SubmitRebalancePlan` for the same `plan_id` must be safe (`execution` dedupes on `plan_id`).
8. **No event emission on empty plans** — if the trigger evaluates and no actions are warranted, drop the plan; do not pollute the bus.

## Acceptance Criteria (testable)

1. Given a `PortfolioTargetGenerated` where every `WeightDrift < 0.25` and no `open`/`close` actions, when `EvaluateDriftOnNewTarget` runs, then no `RebalanceTriggered` is raised and no `RebalancePlanProposed` is emitted.
2. Given a held ticker with `current_weight=0.10` and `target_weight=0.06` (drift = 67% > 25%), when `EvaluateDriftOnNewTarget` runs, then a `RebalancePlan` is composed with one `RebalanceAction(side=SELL)` for this ticker and `RebalancePlanProposed` is emitted exactly once.
3. Given a `RiskRegimeChanged(current=capital_preservation)`, when `TriggerOnRegimeChange` runs and the current target violates the new caps, then a defensive-only `RebalancePlan` is composed (only SELL/trim actions, no BUY) and submitted via sync `SubmitRebalancePlan`.
4. Given a single ticker requires `Δ weight = 0.10 × portfolio_value` and `max_order_size = 0.05`, when composing the plan, then this ticker yields **one** action sized `0.05 × portfolio_value` (the next slice is handled on the next pass).
5. Given `SubmitRebalancePlan` returns `OrderRejected` for one of two actions, when handling the response, then the plan is marked partially failed and the next pass's `RebalanceTriggered` includes the failed ticker (bounded retries).
6. Given an `InsiderTransactionRecorded(transaction_type=sell)` for a held ticker by an officer above the significance threshold, when `TriggerOnInsiderSelling` runs, then the resulting plan contains only `SELL` actions for that ticker (defensive-only invariant).
7. Given two `PortfolioTargetGenerated` events with the same `event_id`, when both delivered, then `EvaluateDriftOnNewTarget` runs evaluation exactly once (idempotent).
8. Given `LatestPriceView` for a held ticker is older than the staleness threshold, when computing `WeightDrift`, then the ticker is skipped, a warning is logged, and the plan is composed for the remaining tickers only.
9. Given a `CatalystIdentified` for a non-held ticker, when `TriggerOnCatalystOrConvictionChange` runs, then no `RebalanceTriggered` is raised (non-held tickers route through `portfolio-construction` only).

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | `WeightDrift` definition — relative (`|Δ| / target`) per PRD §4.6, or absolute (`|Δ| > 2%`)? Relative breaks for `target_weight` near 0. | Domain expert | Open (lean: relative with absolute floor for small targets) |
| 2 | Insider-sell significance threshold — shares-only? notional? % of insider's holding? Section 16 routine sales (10b5-1) excluded? | Domain expert | Open |
| 3 | UC-2 wait-for-recompose timeout (default 5s proposed). Acceptable? | Domain expert | Open |
| 4 | UC-7 sync-submit retry policy on `OrderRejected` — N retries on next pass, then halt? Per-action vs. per-plan? | Domain expert | Open |
| 5 | Risk-alert → trigger-type mapping (UC-6) — none of the catalog `RebalanceTrigger` enum values cleanly maps to `risk_alert`. Add a new enum value, or reuse the closest? Either way requires updating ubiquitous-language + event-catalog. | Domain expert + architecture | Open |
| 6 | `PriceMoveExceeded` lookback — 1 trading day per PRD §4.6 implicit? Or intra-day (e.g., last cadence tick)? | Domain expert | Open |
| 7 | Default `limit_price` policy — last × (1 ± N bps)? mid? last? | Domain expert | Open |
| 8 | Should `EarningsEvent` trigger fire pre-earnings (e.g., 1 trading day before) or only on the catalyst event itself? | Domain expert | Open |

## Approval

| Reviewer | Role | Decision | Date |
|----------|------|----------|------|
| TBD | Domain expert | Pending | — |

## Related Docs

| Doc | Path |
|-----|------|
| Source PRD(s) | [../../../../input/prd.md](../../../../input/prd.md), [../../../../input/investment-frameworks.md](../../../../input/investment-frameworks.md) |
| Context Map | [../../../architecture/context-map.md](../../../architecture/context-map.md) |
| Architecture | [../../../architecture/architecture.md](../../../architecture/architecture.md) |
| Ubiquitous Language | [../../../architecture/ubiquitous-language.md](../../../architecture/ubiquitous-language.md) |
| Event Catalog | [../../../architecture/event-catalog.md](../../../architecture/event-catalog.md) |
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0003-orchestrator-and-sync-rebalance.md](../../../architecture/adr/0003-orchestrator-and-sync-rebalance.md)) |
