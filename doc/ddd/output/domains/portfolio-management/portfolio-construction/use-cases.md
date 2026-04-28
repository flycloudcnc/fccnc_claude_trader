# Intent Translation: portfolio-management/portfolio-construction

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.5 Portfolio Construction Agent; §5 Execution Loop; §7 Configuration), `doc/ddd/input/investment-frameworks.md` (frameworks 7 Pabrai, 1 Greenblatt, 8 Marks)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Combine `AlphaCandidate`, `ConvictionAssessment`, and `CatalystAssessment` signals from the signal-intelligence domain with the current `RiskRegime` caps and the user-supplied `Configuration` constraints (`PortfolioSize`, `max_single_position`, `max_sector`, `cash_min`/`cash_max`) to compute a `PortfolioTarget` — the set of `TargetPosition`s the portfolio should hold after the current pass. The target is a desired end-state (weights + per-ticker action); it does **not** issue orders. `rebalancing` consumes the target and decides whether to act.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `portfolio-management/portfolio-construction` | Yes | Owns `PortfolioTarget` aggregate. |
| `signal-intelligence/alpha-generation` | No | Provides `AlphaCandidateGenerated` + `CandidateRankingView`. |
| `signal-intelligence/conviction-sizing` | No | Provides `ConvictionAssessed` + `ConvictionView`. |
| `signal-intelligence/catalyst-timing` | No | Provides `CatalystIdentified` + `CatalystView`. |
| `portfolio-management/risk-regime` | No | Provides `RiskRegimeChanged` + `CurrentRegimeView` (caps). |
| `trade-operations/execution` | No | Provides `AccountPositionView` (current holdings → action classification). |
| `portfolio-management/rebalancing` | No | Consumes `PortfolioTargetGenerated`. |
| `research-validation/backtesting` | No | Replays composition over historical signals. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Portfolio Construction Agent (PRD §4.5) | PortfolioConstructionService | `PortfolioConstructionService` | Existing |
| `target_weight` (PRD §4.5 output) | TargetPosition.target_weight | `TargetPosition.target_weight: Weight` | Existing |
| `action` (PRD §4.5: "increase" / "trim" etc.) | TargetPosition.action | `TargetPosition.action ∈ {open, increase, trim, close, hold}` | Existing |
| Constraints (PRD §4.5: `max_positions`, `max_single_position`, `max_sector`, `cash_min`, `cash_max`) | Constraint VOs (per-context) | `Constraint` | Existing |
| `risk_adjustment` (PRD §4.5 formula) | RiskAdjustment | `RiskAdjustment` | Existing |
| `target_weight = alpha × conviction × catalyst × risk_adjustment` (PRD §4.5 formula) | (Formula encoded in `PortfolioConstructionService.compose`) | — | — |
| Portfolio targets list (PRD §4.5 output) | PortfolioTarget aggregate | `PortfolioTarget` | Existing |

## Use Cases

### UC-1: ComposePortfolioTarget

- **Actor:** Trading Loop orchestrator.
- **Trigger:** Cadence tick — orchestrator step "compose target" begins after `risk-regime` step has run for the current `RunId`.
- **Goal:** Compute a `PortfolioTarget` from the current pass's `AlphaCandidate`s, their `ConvictionAssessment`s, `CatalystAssessment`s, the current `RiskRegimeState`, current `AccountPosition`s, and `Configuration` constraints; emit `PortfolioTargetGenerated`.
- **Primary Aggregate:** `PortfolioTarget` (one per `RunId`).
- **Pre-conditions:** `CandidateRankingView`, `ConvictionView`, `CatalystView`, `CurrentRegimeView`, `AccountPositionView` populated; `Configuration` loaded.
- **Outcome (success):** `PortfolioTarget` persisted with `TargetPosition`s satisfying every constraint; `PortfolioTargetGenerated` emitted exactly once per `RunId`.
- **Outcome (failure):** No candidates available → emit empty target with `cash_target = 1.0` and `positions = []` (degenerate but valid). Constraint conflict detected (e.g., `max_single_position` × `PortfolioSize` < total committed weight) → composition fails with `ConstraintConflictError`; no event emitted; `monitoring` raises a `warning` alert. Missing regime view → reuse last-known regime, log `RegimeViewStale` (no event suppression).
- **Invariants Touched:** `Σ target_weight + cash_target == 1`; `count(positions) ≤ PortfolioSize`; `target_weight ≤ min(max_single_position, RiskRegimeState.max_position)` per position; `Σ target_weight per sector ≤ max_sector`; `cash_min ≤ cash_target ≤ cash_max`; `Σ target_weight ≤ RiskRegimeState.max_equity_exposure`; one `PortfolioTarget` per `RunId`.
- **Cross-context Calls:** Reads `CandidateRankingView`, `ConvictionView`, `CatalystView` (signal-intel), `CurrentRegimeView` (risk-regime), `AccountPositionView` (execution); emits `PortfolioTargetGenerated` to `rebalancing` and `backtesting`.

### UC-2: RecomposeOnRegimeChange

- **Actor:** `portfolio-construction` event subscriber.
- **Trigger:** `RiskRegimeChanged` from `risk-regime` (any transition).
- **Goal:** Re-compose the current `PortfolioTarget` immediately under the new regime caps, without waiting for the next cadence tick. Emit a refreshed `PortfolioTargetGenerated`.
- **Primary Aggregate:** `PortfolioTarget` (current pass).
- **Pre-conditions:** A `PortfolioTarget` for the current `RunId` exists (or none exists yet — see Outcome). Latest signal views available.
- **Outcome (success):** `PortfolioTarget` upserted under new caps; `PortfolioTargetGenerated` emitted with new `event_id` and the same `target_id` (same aggregate, version-incremented). If no current target exists yet, the regime change is buffered and applied on the next UC-1 run.
- **Outcome (failure):** Same as UC-1 (constraint conflict → no event; missing inputs → reuse last target). Dead-letter on retry exhaustion → `monitoring` raises `RiskAlertRaised(severity=warning)`.
- **Invariants Touched:** Same as UC-1; additionally, recompose must not lower `count(positions)` below an explicit floor without an explicit close action per position (no implicit "drop" of positions — every removed position becomes a `close` action so `rebalancing` sees the intent).
- **Cross-context Calls:** Subscribes to `RiskRegimeChanged`; emits `PortfolioTargetGenerated`.

### UC-3: ClassifyActionsAgainstCurrentPositions

- **Actor:** Internal step of UC-1 / UC-2 (not standalone).
- **Trigger:** During composition, after target weights are computed.
- **Goal:** For each `TargetPosition`, classify the `action` (`open` / `increase` / `trim` / `close` / `hold`) by comparing to the corresponding `AccountPosition` weight (or absence) from `AccountPositionView`.
- **Primary Aggregate:** `PortfolioTarget` (mutated in-flight).
- **Pre-conditions:** Target weights computed; `AccountPositionView` available.
- **Outcome (success):** Every `TargetPosition` carries a deterministic `action`; held tickers absent from the new target receive `action=close` with `target_weight=0`.
- **Outcome (failure):** `AccountPositionView` stale beyond a configured threshold (default 60s) → composition fails fast with `StalePositionsError` (do not generate misleading actions); `monitoring` raises `warning`.
- **Invariants Touched:** `action == open` ⇔ `current_weight == 0 ∧ target_weight > 0`; `action == close` ⇔ `current_weight > 0 ∧ target_weight == 0`; `action == hold` ⇔ `|target_weight − current_weight| ≤ hold_threshold`; every held ticker not in the new target appears in the target with `action=close`.
- **Cross-context Calls:** Reads `AccountPositionView`.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `Σ target_weight + cash_target == 1` (within float tolerance) | `PortfolioTarget` | New |
| `count(positions where target_weight > 0) ≤ PortfolioSize` | `PortfolioTarget` | New |
| `target_weight ≤ min(Configuration.max_single_position, RiskRegimeState.max_position)` per position | `TargetPosition` | New |
| `Σ target_weight per sector ≤ Configuration.max_sector` | `PortfolioTarget` | New |
| `Configuration.cash_min ≤ cash_target ≤ Configuration.cash_max` | `PortfolioTarget` | New |
| `Σ target_weight ≤ RiskRegimeState.max_equity_exposure` | `PortfolioTarget` | New |
| One `PortfolioTarget` per `RunId` (regime-driven recompose increments aggregate version, same `target_id`) | `PortfolioTarget` | New |
| Held tickers dropped from target appear with `action=close, target_weight=0` (no implicit drops) | `PortfolioTarget` | New |
| `action` derived deterministically from `(current_weight, target_weight)` | `TargetPosition` | New |
| Composition is idempotent on `(RunId, regime_version, signal_versions)` | `PortfolioConstructionService` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `alpha-generation` | Event: `AlphaCandidateGenerated` + read-model `CandidateRankingView` | Alpha component of formula. |
| Inbound | `conviction-sizing` | Event: `ConvictionAssessed` + read-model `ConvictionView` | Conviction component (`target_weight_range`). |
| Inbound | `catalyst-timing` | Event: `CatalystIdentified` + read-model `CatalystView` | Catalyst component / timing bias. |
| Inbound | `risk-regime` | Event: `RiskRegimeChanged` + read-model `CurrentRegimeView` | Cap inputs and reactive recompose. |
| Inbound | `execution` | Read-model: `AccountPositionView` | Current weights → `action` classification. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `PortfolioSize`, `MaxDrawdown`, `RiskTolerance` shape constraints. |
| Outbound | `rebalancing` | Event: `PortfolioTargetGenerated` | Drift evaluation input. |
| Outbound | `backtesting` | Event: `PortfolioTargetGenerated` (backtest bus) | Replay verification. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Composition formula (`alpha × conviction × catalyst × risk_adjustment`) and normalization | `portfolio-construction` | TBD |
| `RiskAdjustment` lookup (regime → multiplier) | `portfolio-construction` | TBD |
| Constraint validator (sector caps, single-position caps, cash bands) | `portfolio-construction` | TBD |
| Action classifier (UC-3) | `portfolio-construction` | TBD |
| Sector-mapping data source (GICS) | `portfolio-construction` | TBD |

## Constraints & Prohibited Actions

1. **No order writes** — `portfolio-construction` produces a `PortfolioTarget` only. Order generation lives in `execution` (via `rebalancing`'s `RebalancePlan`).
2. **No direct DB reads** of other contexts' tables (HR-1) — only documented read-models and bus events.
3. **Constraints are hard, not soft** — a target that violates `max_single_position`, `max_sector`, `cash_min`/`cash_max`, or `RiskRegimeState` caps must fail composition (`ConstraintConflictError`); no silent clipping.
4. **No implicit position drops** — every held ticker not selected for the new target must appear with `action=close, target_weight=0` so `rebalancing` sees the explicit intent (anchored by ubiquitous-language `TargetPosition.action`).
5. **Idempotent on inputs** — same `(RunId, regime version, signal versions)` must produce the same target (`temperature=0` if any LLM is used; v1 composition is rule-based per PRD §4.5 formula).
6. **Cost discipline** — composition is rule-based in v1; if an LLM is later introduced for synthesis (e.g., reason narration), use Opus only and prompt-cache (architecture.md §Key Constraints).

## Acceptance Criteria (testable)

1. Given populated views for 30 candidates, `PortfolioSize=20`, `max_single_position=0.10`, `max_sector=0.25`, `cash_min=0.05`, `cash_max=0.50`, and `RiskRegimeState.max_equity_exposure=0.75`, when `ComposePortfolioTarget` runs, then exactly one `PortfolioTargetGenerated` is emitted with `count(positions where target_weight > 0) ≤ 20`, every `target_weight ≤ 0.10`, every sector `≤ 0.25`, `Σ target_weight ≤ 0.75`, and `Σ target_weight + cash_target == 1`.
2. Given a candidate with `ConvictionAssessment.target_weight_range=[0.03, 0.06]` and `CatalystAssessment.entry_action=buy_now`, when composing, then its `target_weight` is within `[0.03, 0.06]` (or `0` if cap-bound, never above the upper bound).
3. Given a held ticker absent from the new candidate set, when composing, then it appears in `PortfolioTarget.positions` with `action=close, target_weight=0`.
4. Given `RiskRegimeChanged(current=capital_preservation, max_equity_exposure=0.20)` while a target with `Σ target_weight=0.70` exists, when `RecomposeOnRegimeChange` handles it, then a refreshed `PortfolioTargetGenerated` is emitted with `Σ target_weight ≤ 0.20` and the same `target_id`, version-incremented.
5. Given `AccountPositionView.as_of` is older than 60s, when composing, then composition raises `StalePositionsError`, no event is emitted, and a `warning` alert is raised.
6. Given two recomposes with identical inputs, when both run, then the resulting `target_weight` per ticker is identical (deterministic).
7. Given a candidate set whose minimum required weights sum to > `RiskRegimeState.max_equity_exposure` under all constraints, when composing, then `ConstraintConflictError` is raised, no event is emitted, and `monitoring` raises a `warning`.
8. Given two `RiskRegimeChanged` events with the same `event_id`, when both delivered, then `RecomposeOnRegimeChange` runs the recompose exactly once (idempotent on `(event_name, event_id)`).

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Composition formula — PRD §4.5 lists `alpha × conviction × catalyst × risk_adjustment`. Confirm: is this a pure multiplicative score that is then normalized to weights summing to `≤ max_equity_exposure − cash_target`, or is each factor a separate clamp? | Domain expert | Open |
| 2 | `RiskAdjustment` table — what multiplier per regime (`risk_on`/`neutral`/`risk_off`/`capital_preservation`)? | Domain expert | Open |
| 3 | `hold_threshold` for action classification (e.g., 0.5% absolute drift)? Without this, every recompose flips many `hold` → `increase`/`trim`. | Domain expert | Open |
| 4 | Sector source — GICS via vendor field on `FundamentalSnapshot`? Alternative? | Domain expert | Open |
| 5 | Recompose-on-regime semantics — same `target_id` (preferred) vs. new `target_id`? Same id keeps `rebalancing` grounded on a single in-flight target per pass. | Domain expert | Open (lean: same `target_id`, version-incremented) |
| 6 | Behavior when a constraint is unsatisfiable — fail-fast (current proposal) vs. relax `cash_max` first vs. relax `max_sector`? | Domain expert | Open |
| 7 | Catalyst influence — is `entry_action=wait` a reason to set `target_weight=0` for `open` actions, or only to delay an `increase`? | Domain expert | Open |

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
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0002-configuration-shared-kernel.md](../../../architecture/adr/0002-configuration-shared-kernel.md), [0003-orchestrator-and-sync-rebalance.md](../../../architecture/adr/0003-orchestrator-and-sync-rebalance.md)) |
