# Intent Translation: trade-operations/monitoring

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.8 Monitoring Agent; §9 Critical Risks), `doc/ddd/input/investment-frameworks.md` (frameworks 8 Marks, 7 Pabrai)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Compute periodic `PortfolioHealthSnapshot`s (drawdown, 30-day volatility, sector exposure, cash, benchmark return) from `AccountPositionView` + `LatestPriceView`, and raise `RiskAlert`s when health metrics breach thresholds (sector concentration, drawdown breach, repeated order rejections, stale price data, fill latency, loop overruns). Alerts close the feedback loop into `risk-regime` (potential regime downgrade) and `rebalancing` (defensive plan trigger). `monitoring` is **observational, not authoritative** — it never writes to portfolio state or submits orders; its outputs are signals for other contexts to act on.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `trade-operations/monitoring` | Yes | Owns `PortfolioHealthSnapshot` + `RiskAlert` aggregates. |
| `trade-operations/execution` | No | Source of `OrderSubmitted`, `OrderFilled`, `OrderRejected`, `PositionUpdated`; provides `AccountPositionView`. |
| `signal-intelligence/market-data` | No | Source of `PriceSnapshotIngested`; provides `LatestPriceView` for valuation + staleness check. |
| `portfolio-management/risk-regime` | No | Consumer of `RiskAlertRaised` (regime downgrade). |
| `portfolio-management/rebalancing` | No | Consumer of `RiskAlertRaised` (defensive trigger); also publishes `RebalancePlanProposed` which `monitoring` records for audit. |
| `orchestrator` | No | Source of `LoopOverrun` (operational event). |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Monitoring Agent (PRD §4.8) | MonitoringService | `MonitoringService` | Existing |
| Drawdown (PRD §4.8) | Drawdown | `Drawdown` | Existing |
| Volatility (PRD §4.8) | Volatility (30d, configurable lookback) | `volatility_30d: decimal` | Existing |
| Sector exposure (PRD §4.8) | SectorExposure | `SectorExposure` | Existing |
| Cash (PRD §4.8) | Cash | `cash: Money` | Existing |
| Benchmark return (PRD §4.8) | BenchmarkReturn (30d) | `benchmark_return_30d: decimal` | Existing |
| `risk_status` (PRD §4.8 output) | AlertSeverity (`info`/`warning`/`critical`) | `AlertSeverity` | Existing |
| `issue` / `action` (PRD §4.8 output) | RiskAlert.description / recommended_action | `RiskAlert` | Existing |
| Health snapshot (system-level) | PortfolioHealthSnapshot | `PortfolioHealthSnapshot` | Existing |

## Use Cases

### UC-1: TakeHealthSnapshot

- **Actor:** Trading Loop orchestrator.
- **Trigger:** Cadence tick — orchestrator step "monitor" runs after `execution` step (or unconditionally per cadence if no execution occurred).
- **Goal:** Compute a `PortfolioHealthSnapshot` from current `AccountPositionView` + `LatestPriceView` + `Configuration` (`MaxDrawdown`, `Horizon`); persist; emit `PortfolioHealthSnapshotTaken` (low-criticality archival event per event-catalog).
- **Primary Aggregate:** `PortfolioHealthSnapshot`.
- **Pre-conditions:** `AccountPositionView` populated; `LatestPriceView` available for every held ticker (or staleness flagged per UC-3).
- **Outcome (success):** Snapshot persisted with `total_value`, `cash`, `drawdown` (vs. peak over a configurable lookback, default since-process-start with persistent peak), `volatility_30d` (annualized stdev of daily returns), `sector_exposure` (map<sector, decimal>), and optional `benchmark_return_30d`. `PortfolioHealthSnapshotTaken` emitted; `PortfolioHealthView` projection refreshed.
- **Outcome (failure):** Stale price data for ≥ 1 held ticker → snapshot still computed, mark affected positions with `valuation_stale=true`; trigger UC-3 (UC-3 raises a `stale_price` alert if ≥ N tickers stale).
- **Invariants Touched:** `total_value = Σ (quantity × last_price) + cash`; `drawdown = max(0, (peak − total_value) / peak)`; `Σ sector_exposure ≤ 1.0` (with rounding tolerance); volatility uses ≥ 20 trading days of data or is null.
- **Cross-context Calls:** Reads `AccountPositionView`, `LatestPriceView`; emits `PortfolioHealthSnapshotTaken` (read-model only — no subscribers per event-catalog).

### UC-2: RaiseConcentrationOrDrawdownAlert

- **Actor:** Internal — invoked at the end of UC-1.
- **Trigger:** Snapshot just persisted.
- **Goal:** Evaluate threshold rules — `max_sector` exposure breach (versus `Configuration.max_sector` from `portfolio-construction` constraints, mirrored in `MonitoringConfig`), `drawdown` breach (versus `Configuration.MaxDrawdown`); raise one `RiskAlert` per distinct breach with `category` and `severity`. Emit `RiskAlertRaised` per new alert (no duplicate alerts within a configurable cooldown).
- **Primary Aggregate:** `RiskAlert`.
- **Pre-conditions:** Snapshot persisted; thresholds loaded from `Configuration`.
- **Outcome (success):** Per breach: if no open `RiskAlert(category=...)` exists in cooldown window (default 1 cadence tick), persist a new alert and emit `RiskAlertRaised`. If an open alert already exists for the same category, update `affected_tickers` and `description` (no re-emit).
- **Outcome (failure):** Threshold table missing for a category → log warning; do not silently skip.
- **Invariants Touched:** `severity ∈ {info, warning, critical}`; `category` from a finite enum (`sector_concentration`, `drawdown_breach`, `repeated_rejections`, `stale_price`, `fill_latency`, `loop_overrun`); one open `RiskAlert` per `(category, scope)` at a time within cooldown; `severity=critical` triggers downstream-halt semantics in `risk-regime` (per event-catalog `RiskRegimeChanged` failure handling).
- **Cross-context Calls:** Emits `RiskAlertRaised` to `risk-regime` and `rebalancing`.

### UC-3: DetectStalePriceData

- **Actor:** `monitoring` event subscriber.
- **Trigger:** Lack of `PriceSnapshotIngested` for a held ticker over the configured staleness window (default: > 24h on a market day for daily cadence; > 5 min on intra-day cadence). Evaluated at every UC-1 snapshot.
- **Goal:** Raise `RiskAlertRaised(category=stale_price, severity=warning, affected_tickers=[…])` so `rebalancing` can defer using stale prices and `risk-regime` can downgrade if multiple tickers stale.
- **Primary Aggregate:** `RiskAlert`.
- **Pre-conditions:** Held positions known; `LatestPriceView.as_of` available per ticker.
- **Outcome (success):** Alert raised with affected tickers; `severity=critical` if **all** held tickers are stale (data feed outage).
- **Outcome (failure):** Market is closed → suppress (do not raise during scheduled closures).
- **Invariants Touched:** Market-hours-aware suppression; alert frequency capped (one alert per affected ticker per cadence tick).
- **Cross-context Calls:** Emits `RiskAlertRaised`; reads `LatestPriceView` + market calendar.

### UC-4: TrackRejectionsAndFillLatency

- **Actor:** `monitoring` event subscriber.
- **Trigger:** `OrderRejected`, `OrderSubmitted`, `OrderFilled` from `execution`.
- **Goal:** Maintain rolling counters (default lookback: 1 cadence tick + 24h) of rejections and submit-to-fill latency; raise `RiskAlertRaised(category=repeated_rejections, severity=warning)` after N rejections in window (default N=3); raise `RiskAlertRaised(category=fill_latency, severity=warning)` if median submit-to-fill ≥ threshold (TBD with domain expert).
- **Primary Aggregate:** `RiskAlert` (created on threshold breach); rolling state held in `MonitoringService` (read via `OpenAlertsView`).
- **Pre-conditions:** Subscribed to execution events; counters initialized.
- **Outcome (success):** Counters updated on every event; alert raised at most once per cooldown per category.
- **Outcome (failure):** Dead-letter event from execution (e.g., `OrderRejected` not delivered) → log warning; counters may under-count temporarily; reconcile on next successful delivery.
- **Invariants Touched:** Counter increments are idempotent on `(event_name, event_id)` (HR-7); rolling-window expiry is monotonic.
- **Cross-context Calls:** Subscribes to `OrderSubmitted`, `OrderFilled`, `OrderRejected`; emits `RiskAlertRaised`.

### UC-5: RecordLoopOverrun

- **Actor:** `monitoring` event subscriber.
- **Trigger:** `LoopOverrun` (operational event from `orchestrator`) when a Trading-Loop pass exceeds 80% of cadence interval (architecture.md §Key Constraints #1).
- **Goal:** Log the overrun; raise `RiskAlertRaised(category=loop_overrun, severity=warning)` after N overruns (default N=3) in a sliding window.
- **Primary Aggregate:** `RiskAlert`.
- **Pre-conditions:** Subscribed to `LoopOverrun`.
- **Outcome (success):** Counter incremented; alert raised on threshold; on alert, recommended_action mentions the `slowest_step` from the event payload.
- **Outcome (failure):** Counter wraps incorrectly across process restart → restart resets counter (acceptable in v1).
- **Invariants Touched:** One open `loop_overrun` alert at a time; closes when N consecutive on-time passes occur.
- **Cross-context Calls:** Subscribes to `LoopOverrun`; emits `RiskAlertRaised`.

### UC-6: RecordRebalancePlanForAudit

- **Actor:** `monitoring` event subscriber.
- **Trigger:** `RebalancePlanProposed` from `rebalancing`.
- **Goal:** Persist the plan summary to the audit log (append-only `audit/` jsonl) for traceability — does not raise alerts in v1.
- **Primary Aggregate:** N/A — audit-only side effect; no aggregate state changed.
- **Pre-conditions:** Audit log writable.
- **Outcome (success):** Plan summary appended (`plan_id`, `target_id`, `actions[]`, `run_id`, timestamp).
- **Outcome (failure):** Audit log write fails → process restart per supervisor (audit must not silently drop per architecture.md §Cross-Cutting / Audit).
- **Invariants Touched:** Audit log is append-only; idempotent on `event_id`.
- **Cross-context Calls:** Subscribes to `RebalancePlanProposed`; no event emitted.

### UC-7: ResolveOpenAlert

- **Actor:** `monitoring` self-tick (cadence) or operator command.
- **Trigger:** Periodic re-evaluation of open alerts; or explicit `resolve_alert(alert_id)` operator action.
- **Goal:** Mark an open `RiskAlert` resolved when its threshold is no longer breached for the configured cooldown (e.g., sector exposure back below cap for 2 consecutive snapshots).
- **Primary Aggregate:** `RiskAlert` (terminal state: `RESOLVED`).
- **Pre-conditions:** Open alert exists; threshold check available.
- **Outcome (success):** Alert marked `RESOLVED` with timestamp; **no event emitted in v1** (downstream contexts re-evaluate via the next cadence's `RiskAlertRaised`-or-not). Open question: do `risk-regime` / `rebalancing` need an `AlertResolved` event to re-upgrade?
- **Outcome (failure):** Threshold still breached → alert stays open; counter re-extends cooldown.
- **Invariants Touched:** `RESOLVED` is terminal; no further state changes.
- **Cross-context Calls:** None in v1 (subject to open question #5).

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `total_value = Σ (quantity × last_price) + cash` | `PortfolioHealthSnapshot` | New |
| `drawdown = max(0, (peak − total_value) / peak)` with persisted `peak` | `PortfolioHealthSnapshot` | New |
| `Σ sector_exposure ≤ 1.0` (within rounding tolerance) | `PortfolioHealthSnapshot` | New |
| `volatility_30d` is null until ≥ 20 trading days of returns are available | `PortfolioHealthSnapshot` | New |
| `severity ∈ {info, warning, critical}` (anchor: event-catalog) | `RiskAlert` | New |
| `category` from finite enum (`sector_concentration`, `drawdown_breach`, `repeated_rejections`, `stale_price`, `fill_latency`, `loop_overrun`) | `RiskAlert` | New |
| One open `RiskAlert` per `(category, scope)` at a time within cooldown window | `MonitoringService` | New (anti-noise) |
| `severity=critical` for execution-side dead-letters (anchor: event-catalog Dead Letter row) | `MonitoringService` | New |
| `monitoring` is observational only — never submits orders, never writes to other aggregates | `MonitoringService` | New (architectural) |
| Counter increments are idempotent on `(event_name, event_id)` | `MonitoringService` | New (HR-7 anchor) |
| Stale-price alerts are market-hours-aware (suppressed during scheduled closures) | `MonitoringService` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `execution` | Read-model: `AccountPositionView`; events `OrderSubmitted`, `OrderFilled`, `OrderRejected`, `PositionUpdated` | Snapshot inputs + rejection/latency tracking. |
| Inbound | `market-data` | Read-model: `LatestPriceView`; event `PriceSnapshotIngested` | Valuation + staleness detection. |
| Inbound | `rebalancing` | Event: `RebalancePlanProposed` | Audit-only persistence (UC-6). |
| Inbound | `orchestrator` | Event: `LoopOverrun` | Overrun tracking (UC-5). |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `MaxDrawdown`, sector caps mirror, lookbacks. |
| Outbound | `risk-regime` | Event: `RiskAlertRaised` | Regime downgrade input. |
| Outbound | `rebalancing` | Event: `RiskAlertRaised` | Defensive plan trigger. |
| Outbound | (read-model only) | `PortfolioHealthView`, `OpenAlertsView` | Operator dashboard / debugging. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Health metric definitions (drawdown, volatility window, peak persistence) | `monitoring` | TBD |
| Threshold table (per category → severity) | `monitoring` | TBD |
| Cooldown / hysteresis policy | `monitoring` | TBD |
| Sector mapping mirror (must agree with `portfolio-construction` source) | `monitoring` | TBD |
| Audit log writer | `monitoring` × architecture | TBD |

## Constraints & Prohibited Actions

1. **Observational only** — `monitoring` MUST NOT submit orders, write to `AccountPosition`, or modify `PortfolioTarget`. Outputs are alerts and read-models.
2. **No event emission on no-op** — `PortfolioHealthSnapshotTaken` always emits (archival), but `RiskAlertRaised` emits only on a transition into a breach within cooldown.
3. **Cooldown to suppress noise** — repeated breaches within cooldown extend the open alert; do not spam the bus.
4. **Market-hours awareness** — `stale_price` alerts respect market calendar (no alerts during scheduled closures).
5. **Idempotency** — every subscriber dedupes on `(event_name, event_id)` (HR-7).
6. **No direct DB reads** of other contexts' tables (HR-1) — only documented read-models and bus events.
7. **Audit log is append-only** (architecture.md §Cross-Cutting / Audit) — UC-6 must not edit prior entries.
8. **Sector mapping must agree with `portfolio-construction`** — if they disagree, alerts will fire on phantom breaches; share the source via a read-model or shared module.

## Acceptance Criteria (testable)

1. Given an `AccountPositionView` with two positions and matching `LatestPriceView`, when `TakeHealthSnapshot` runs, then a `PortfolioHealthSnapshot` is persisted with `total_value = Σ(quantity × last_price) + cash`, and `PortfolioHealthSnapshotTaken` is emitted exactly once.
2. Given a snapshot with sector A exposure = 0.30 and `Configuration.max_sector = 0.25`, when `RaiseConcentrationOrDrawdownAlert` runs and no open `sector_concentration` alert exists, then exactly one `RiskAlertRaised(category=sector_concentration, severity=warning, affected_tickers=[…sector A tickers…])` is emitted.
3. Given the same condition repeats on the next cadence tick within cooldown, when `RaiseConcentrationOrDrawdownAlert` runs, then no new `RiskAlertRaised` is emitted (cooldown active).
4. Given `drawdown = 0.18` and `Configuration.MaxDrawdown = 0.15`, when evaluating, then a `RiskAlertRaised(category=drawdown_breach, severity=critical)` is emitted.
5. Given 3 `OrderRejected` events for the same plan within the rejection window, when `TrackRejectionsAndFillLatency` evaluates, then a `RiskAlertRaised(category=repeated_rejections, severity=warning)` is emitted exactly once until cooldown expires.
6. Given `LatestPriceView.as_of` for all held tickers is older than the staleness window during market hours, when `DetectStalePriceData` runs, then `RiskAlertRaised(category=stale_price, severity=critical, affected_tickers=[all])` is emitted.
7. Given the same stale condition during a scheduled market closure, when `DetectStalePriceData` runs, then no event is emitted.
8. Given `LoopOverrun` fires 3 times in the configured window, when `RecordLoopOverrun` evaluates the third, then `RiskAlertRaised(category=loop_overrun, severity=warning)` is emitted with `recommended_action` mentioning `slowest_step` from the latest event.
9. Given `RebalancePlanProposed` for `plan_id=X`, when `RecordRebalancePlanForAudit` runs, then a single audit-log line is appended with `plan_id`, `target_id`, `actions[]`, `run_id`, timestamp; no event is emitted.
10. Given two `OrderRejected` with the same `event_id`, when both delivered, then the rejection counter is incremented exactly once (idempotent).
11. Given an open `sector_concentration` alert and the next snapshot shows exposure back to 0.20 for 2 consecutive cadence ticks, when `ResolveOpenAlert` runs, then the alert is marked `RESOLVED` (no event emitted in v1).

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Drawdown peak — since-process-start with persistence, or rolling N-day peak (e.g., 252d)? | Domain expert | Open |
| 2 | Volatility window — 30d (PRD §4.8 implicit)? Annualized? Daily-return-based? | Domain expert | Open |
| 3 | Threshold table per category — full list of `(category, severity, threshold)` tuples (e.g., `sector_concentration > 0.25 → warning`, `drawdown_breach > MaxDrawdown → critical`). | Domain expert | Open |
| 4 | Cooldown duration per category (e.g., 1 cadence tick? N hours?). Avoids alert flapping. | Domain expert | Open |
| 5 | Do we need an `AlertResolved` domain event? Without it, downstream `risk-regime` cannot re-upgrade automatically — it will only revisit on next cadence's `RecomputeRegime`. Acceptable for v1? | Architecture + domain expert | Open |
| 6 | Benchmark for `benchmark_return_30d` — SPY? VTI? Per `Configuration`? Required or optional? | Domain expert | Open |
| 7 | Fill-latency threshold — median submit-to-fill > X seconds raises alert. Reasonable X for paper trading vs. live? | Domain expert | Open |
| 8 | Sector source — must mirror `portfolio-construction` exactly. Should we share a `SectorClassifier` module via shared kernel, or is each context allowed its own? | Architecture | Open (lean: shared module) |

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
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0002-configuration-shared-kernel.md](../../../architecture/adr/0002-configuration-shared-kernel.md)) |
