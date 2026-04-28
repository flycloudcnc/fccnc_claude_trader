# Intent Translation: portfolio-management/risk-regime

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.4 Risk Regime Agent), `doc/ddd/input/investment-frameworks.md` (frameworks 7 Pabrai, 8 Marks, 10 Burry)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Maintain the portfolio-wide `RiskRegime` (`risk_on` / `neutral` / `risk_off` / `capital_preservation`) and the exposure caps it implies (`max_equity_exposure`, `cash_target`, `max_position`). Recompute on cadence and on inbound `RiskAlertRaised` from `monitoring`. Publish `RiskRegimeChanged` only when the regime actually changes — downstream construction and rebalancing react to regime transitions, not redundant recompute output.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `portfolio-management/risk-regime` | Yes | Owns `RiskRegimeState` aggregate. |
| `signal-intelligence/market-data` | No | Provides `LatestPriceView` (breadth, volatility series). |
| `trade-operations/monitoring` | No | Source of `RiskAlertRaised`. |
| `portfolio-management/portfolio-construction` | No | Consumes `RiskRegimeChanged` and reads `CurrentRegimeView` for caps. |
| `portfolio-management/rebalancing` | No | `RiskRegimeChanged` is a `RebalanceTrigger`. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Risk Regime Agent (PRD §4.4) | RiskRegimeService | `RiskRegimeService` | Existing |
| Regime (PRD §4.4 enum) | RiskRegime (enum) | `RiskRegime` | Existing |
| `max_equity_exposure` | (Field on `RiskRegimeState`) | `max_equity_exposure: Percent` | Existing |
| `cash_target` | (Field on `RiskRegimeState`) | `cash_target: Percent` | Existing |
| `max_position` | (Field on `RiskRegimeState`) | `max_position: Weight` | Existing |
| Margin of safety (Pabrai) / cycles (Marks) | (Inputs to regime model — narrative inputs to LLM) | — | — |

## Use Cases

### UC-1: RecomputeRegime

- **Actor:** Trading Loop orchestrator.
- **Trigger:** Cadence tick, **before** `portfolio-construction` step.
- **Goal:** Compute the current `RiskRegime` and its caps from breadth/volatility series + `Configuration` (`RiskTolerance`, `MaxDrawdown`, `Horizon`); persist `RiskRegimeState`; emit `RiskRegimeChanged` only on transition.
- **Primary Aggregate:** `RiskRegimeState` (single instance, version-incremented on each change).
- **Pre-conditions:** `LatestPriceView` populated; `Configuration` loaded; previous `RiskRegimeState` accessible.
- **Outcome (success):** State updated; `RiskRegimeChanged` emitted **iff** `current_regime != previous_regime` (matches event-catalog trigger).
- **Outcome (failure):** Insufficient data → keep last regime, log `RegimeRecomputeSkipped`. LLM parse error after retries → keep last regime, `AgentParseError` logged; `monitoring` may raise an alert.
- **Invariants Touched:** `current_regime ∈ {risk_on, neutral, risk_off, capital_preservation}`; `0 ≤ cash_target ≤ 1`; `0 ≤ max_equity_exposure ≤ 1`; `0 ≤ max_position ≤ max_equity_exposure`; transition emits exactly one event.
- **Cross-context Calls:** Reads `LatestPriceView` from `market-data`; emits `RiskRegimeChanged` to `portfolio-construction` and `rebalancing`.

### UC-2: DowngradeRegimeOnRiskAlert

- **Actor:** `risk-regime` event subscriber.
- **Trigger:** `RiskAlertRaised` from `monitoring` with `severity ∈ {warning, critical}`.
- **Goal:** Consider an immediate regime downgrade (e.g., `neutral → risk_off`, `risk_off → capital_preservation`) without waiting for the next cadence tick. Emit `RiskRegimeChanged` if the alert justifies a transition.
- **Primary Aggregate:** `RiskRegimeState`.
- **Pre-conditions:** Existing `RiskRegimeState` exists.
- **Outcome (success):** Downgrade applied if alert category and severity meet downgrade rules; `RiskRegimeChanged` emitted on actual transition. Otherwise no-op (alert acknowledged but regime unchanged).
- **Outcome (failure):** Dead-letter on retry exhaustion → orchestrator halts loop and `monitoring` raises `RiskAlertRaised(severity=critical)` (per event-catalog `RiskRegimeChanged` failure handling).
- **Invariants Touched:** Downgrade is monotonic within a single alert handling (never upgrade in response to an alert); state version is incremented on every change to support FIFO semantics.
- **Cross-context Calls:** Subscribes to `RiskAlertRaised`; emits `RiskRegimeChanged`.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `current_regime ∈ {risk_on, neutral, risk_off, capital_preservation}` | `RiskRegimeState` | New |
| `0 ≤ cash_target ≤ 1` | `RiskRegimeState` | New |
| `0 ≤ max_equity_exposure ≤ 1` | `RiskRegimeState` | New |
| `0 ≤ max_position ≤ max_equity_exposure` | `RiskRegimeState` | New |
| `RiskRegimeChanged` emitted only on `current_regime != previous_regime` | `RiskRegimeService` | New (anchor: event-catalog) |
| Alert-driven downgrade never upgrades the regime | `RiskRegimeService` (UC-2) | New |
| One `RiskRegimeState` exists at any time (singleton aggregate) | `RiskRegimeState` | New |
| LLM output (if used) must conform to `RiskRegimeState` JSON Schema (ADR-0005) | `RiskRegimeService` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `market-data` | Read-model: `LatestPriceView` (breadth + volatility series) | Regime computation. |
| Inbound | `monitoring` | Event: `RiskAlertRaised` | Alert-driven downgrade. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `RiskTolerance`, `MaxDrawdown`, `Horizon` shape regime thresholds. |
| Outbound | `portfolio-construction` | Event: `RiskRegimeChanged` + read-model `CurrentRegimeView` | Apply caps in next composition. |
| Outbound | `rebalancing` | Event: `RiskRegimeChanged` | Triggers immediate rebalance evaluation. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Regime model (rule-based, LLM-assisted, or hybrid) | `risk-regime` | TBD |
| Cap-derivation table (regime → caps) | `risk-regime` | TBD |
| Alert-to-downgrade mapping | `risk-regime` | TBD |

## Constraints & Prohibited Actions

1. **Strict JSON only** for any LLM-driven regime classifier (ADR-0005). Fallback rule-based path is allowed if it returns the same `RiskRegimeState` schema.
2. **No direct portfolio writes** — `risk-regime` informs caps; `portfolio-construction` applies them.
3. **No live order writes**.
4. **No event emission without transition** — recomputes that produce the same regime do not raise `RiskRegimeChanged` (suppresses noise; matches event-catalog trigger).
5. **Alert-driven downgrades are monotonic** — UC-2 may downgrade but never upgrade; upgrades only happen via cadence-driven UC-1.
6. **Caps must be internally consistent** — `cash_target + max_equity_exposure ≤ 1` (no over-allocation); `max_position ≤ max_equity_exposure`.

## Acceptance Criteria (testable)

1. Given `RiskRegimeState.current_regime = neutral` and a recompute that yields `neutral`, when `RecomputeRegime` runs, then no `RiskRegimeChanged` event is emitted.
2. Given `current_regime = neutral` and a recompute that yields `risk_off`, when `RecomputeRegime` runs, then exactly one `RiskRegimeChanged(previous=neutral, current=risk_off)` is emitted with consistent caps.
3. Given the recompute yields caps with `cash_target + max_equity_exposure > 1`, when persistence is attempted, then the operation fails with an invariant error and no event is emitted.
4. Given a `RiskAlertRaised(severity=critical, category=drawdown_breach)`, when `DowngradeRegimeOnRiskAlert` runs and the rule maps to `capital_preservation`, then `RiskRegimeChanged` is emitted with `current_regime=capital_preservation` and `max_equity_exposure` reduced accordingly.
5. Given a `RiskAlertRaised(severity=warning)` for a category not mapped to a downgrade, when handled, then no event is emitted (acknowledged-only).
6. Given the LLM (if used) returns malformed JSON twice, when handled, then the previous `RiskRegimeState` is retained, `AgentParseError` is logged, and no event is emitted.
7. Given two `RiskAlertRaised` of the same `event_id`, when delivered twice, then `DowngradeRegimeOnRiskAlert` runs the downgrade exactly once (idempotent on `(event_name, event_id)`).

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Which series feed the regime model? PRD §4.4 lists frameworks (Marks/Pabrai/Burry) but no concrete inputs (e.g., SPX 200d MA breadth, VIX, AAII sentiment, credit spreads). | Domain expert | Open |
| 2 | Cap table per regime — PRD §4.4 example shows `neutral → max_equity_exposure=0.75, cash_target=0.15, max_position=0.08`. Need full table for `risk_on`, `risk_off`, `capital_preservation`. | Domain expert | Open |
| 3 | Is the regime classifier LLM-based, rule-based, or hybrid in v1? LLM cost is amortized per cadence tick — likely cheap. | Domain expert | Open |
| 4 | Alert-to-downgrade mapping — which `category` values (`sector_concentration`, `drawdown_breach`, repeated rejections, stale price) map to which downgrade? | Domain expert | Open |
| 5 | Cooldown / hysteresis — should regime transitions require N consecutive recomputes to flip, to avoid flapping? | Domain expert | Open |

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
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0002-configuration-shared-kernel.md](../../../architecture/adr/0002-configuration-shared-kernel.md), [0005-llm-agents-return-strict-json.md](../../../architecture/adr/0005-llm-agents-return-strict-json.md)) |
