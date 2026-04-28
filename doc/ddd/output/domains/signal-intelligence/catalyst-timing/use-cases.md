# Intent Translation: signal-intelligence/catalyst-timing

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.3 Catalyst Timing Agent), `doc/ddd/input/investment-frameworks.md` (frameworks 8 Marks, 10 Burry, 11 Insider Buying)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

For each `AlphaCandidate`, identify whether a near-term catalyst exists (earnings inflection, spinoff, buyback, insider-cluster buy, regulatory change, margin recovery) and recommend an `EntryAction` (`buy_now` / `wait` / `pass`). Also detects insider-buying clusters from `InsiderTransactionRecorded` events. LLM-driven with strict JSON (ADR-0005).

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `signal-intelligence/catalyst-timing` | Yes | Owns `CatalystAssessment` aggregate and cluster detection. |
| `signal-intelligence/alpha-generation` | No | Source of `AlphaCandidate`s. |
| `signal-intelligence/market-data` | No | Source of `InsiderTransactionRecorded`, fundamentals, earnings dates. |
| `portfolio-management/portfolio-construction` | No | Consumes `CatalystIdentified` for timing weight. |
| `portfolio-management/rebalancing` | No | Catalyst on a held name can trigger re-evaluation. |
| `research-validation/backtesting` | No | Replays catalyst evaluation historically. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Catalyst Timing Agent (PRD §4.3) | CatalystService | `CatalystService` | Existing |
| Catalyst type (PRD §4.3 list) | CatalystType (enum) | `CatalystType` | Existing |
| Catalyst score (PRD §4.3 output) | CatalystScore | `CatalystScore` | Existing |
| Action (`buy_now`, `wait`, `pass`) | EntryAction | `EntryAction` | Existing |
| Mispricing (PRD §4.3 logic) | Mispricing | `Mispricing` | Existing |
| Insider buying cluster (PRD §4.3 + framework 11) | InsiderCluster (3+ insiders / 30 days) | `InsiderCluster` | Existing (defined in `market-data` glossary, scored here) |

## Use Cases

### UC-1: EvaluateCatalystForCandidate

- **Actor:** `catalyst-timing` event subscriber.
- **Trigger:** `AlphaCandidateGenerated` from `alpha-generation`.
- **Goal:** Produce a `CatalystAssessment` with a `CatalystType`, `CatalystScore`, `EntryAction`, and `timeframe_months`. Emit `CatalystIdentified` only if `entry_action ∈ {buy_now, wait}` (per event-catalog §`CatalystIdentified` — `pass` does **not** raise an event).
- **Primary Aggregate:** `CatalystAssessment` (one per `(run_id, ticker)`).
- **Pre-conditions:** Fundamentals + insider views populated for the ticker; earnings calendar available.
- **Outcome (success):** Assessment persisted; conditional event emit on `buy_now` or `wait`.
- **Outcome (failure):** Missing inputs → action `pass` with reason "insufficient data"; no event. LLM parse error after retries → `AgentParseError`, candidate deferred; no event.
- **Invariants Touched:** `CatalystScore ∈ [0, 100]`; `entry_action ∈ {buy_now, wait, pass}`; one `CatalystAssessment` per `(run_id, ticker)`; events emitted only for non-`pass` actions.
- **Cross-context Calls:** Reads `FundamentalsView`, `InsiderClusterView` from `market-data`; emits `CatalystIdentified` to `portfolio-construction` and `rebalancing`.

### UC-2: ReevaluateOnInsiderClusterDetected

- **Actor:** `catalyst-timing` event subscriber.
- **Trigger:** `InsiderTransactionRecorded` from `market-data` for any ticker with an open `CatalystAssessment` in the current `RunId`, OR any held position in `AccountPositionView`.
- **Goal:** Detect cluster formation (≥3 insiders within 30-day window per framework 11), and if a cluster forms, recompute the catalyst and emit a refreshed `CatalystIdentified` (with `catalyst_type=InsiderClusterBuy` if cluster is the dominant signal).
- **Primary Aggregate:** `CatalystAssessment` (existing or new for held position).
- **Pre-conditions:** Cluster threshold defined (≥3 insiders, 30-day window — confirm with domain expert).
- **Outcome (success):** Cluster detected → `CatalystAssessment` upserted; `CatalystIdentified` emitted with new `event_id` only if action or score band changed.
- **Outcome (failure):** Threshold not met → no-op (cluster does not yet exist). LLM error on rescore → keep last-good assessment; warning logged.
- **Invariants Touched:** Cluster definition is stable (3 insiders / 30 days / same ticker / `transaction_type=buy`); cross-pass holds may carry an assessment without an active alpha candidate.
- **Cross-context Calls:** Subscribes to `InsiderTransactionRecorded`; emits `CatalystIdentified` (refreshed, conditional).

### UC-3: ScheduledEarningsEntryReview

- **Actor:** Trading Loop orchestrator.
- **Trigger:** Cadence tick — for each held `AccountPosition` and current candidates, check upcoming earnings within configurable window (default 14 days).
- **Goal:** Identify `EarningsInflection` catalyst opportunities and emit `CatalystIdentified` where appropriate. Also flags pre-earnings hold/exit decisions for `rebalancing`.
- **Primary Aggregate:** `CatalystAssessment`.
- **Pre-conditions:** Earnings calendar available via `market-data` read-model (or vendor field on `FundamentalsView`).
- **Outcome (success):** Assessments produced for tickers entering the earnings window; events emitted as per UC-1's `pass`-vs-event rule.
- **Outcome (failure):** Calendar unavailable → step skipped, warning logged.
- **Invariants Touched:** A held position can have a `CatalystAssessment` even without an `AlphaCandidate` (this BC's coverage extends to held names).
- **Cross-context Calls:** Reads `AccountPositionView` from `execution`, earnings dates from `market-data`; emits `CatalystIdentified`.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `CatalystScore ∈ [0, 100]` | `CatalystAssessment` | New |
| `entry_action ∈ {buy_now, wait, pass}` | `CatalystAssessment` | New |
| One `CatalystAssessment` per `(run_id, ticker)` | `CatalystAssessment` | New |
| `pass` actions do NOT raise `CatalystIdentified` | `CatalystService` | New (anchor to event-catalog) |
| Cluster definition: ≥3 distinct insiders, 30-day window, `transaction_type=buy`, same ticker | `InsiderCluster` (logical, scored here) | New |
| LLM output must conform to `CatalystAssessment` JSON Schema (ADR-0005) | `CatalystService` | New |
| Refresh emits only on changed `entry_action` or score-band crossing | `CatalystService` | New (anti-noise) |
| Held positions may have `CatalystAssessment` without an `AlphaCandidate` | `CatalystAssessment` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `alpha-generation` | Event: `AlphaCandidateGenerated` | Triggers per-candidate evaluation. |
| Inbound | `market-data` | Event: `InsiderTransactionRecorded` | Cluster formation. |
| Inbound | `market-data` | Read-model: `FundamentalsView`, `InsiderClusterView`, earnings dates | Inputs. |
| Inbound | `execution` | Read-model: `AccountPositionView` | Held-name coverage in UC-2 / UC-3. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `Horizon` shapes timeframe selection. |
| Outbound | `portfolio-construction` | Event: `CatalystIdentified` | Timing weight. |
| Outbound | `rebalancing` | Event: `CatalystIdentified` | Drift / re-evaluation trigger on held names. |
| Outbound | `backtesting` | Event: `CatalystIdentified` (backtest bus) | Replay verification. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Cluster detection algorithm | `catalyst-timing` | TBD |
| Catalyst LLM prompt + JSON Schema (one per `CatalystType`?) | `catalyst-timing` | TBD |
| Earnings-window scheduler | `catalyst-timing` | TBD |

## Constraints & Prohibited Actions

1. **Strict JSON only** (ADR-0005). LLM payload must match `CatalystAssessment` schema; free text → `AgentParseError`.
2. **Cluster detection is local to this BC** — `market-data` only emits raw `InsiderTransactionRecorded`; this BC composes the cluster. (Reaffirms `market-data` UC-3 invariant.)
3. **No event emission on `pass`** — keeps the bus quiet for low-signal candidates (anchor: event-catalog payload table).
4. **No live order writes** — `catalyst-timing` informs but does not execute.
5. **Held-name coverage** — held tickers without an active `AlphaCandidate` still receive catalyst evaluation in UC-2 / UC-3; this BC owns that coverage so `rebalancing` does not re-implement catalyst logic.
6. **Determinism per `RunId`** — same inputs must produce the same `entry_action` and `catalyst_score` (LLM `temperature=0`).

## Acceptance Criteria (testable)

1. Given an `AlphaCandidateGenerated` for ticker `XYZ` and complete inputs, when `EvaluateCatalystForCandidate` runs, then a `CatalystAssessment` is persisted; if `entry_action ∈ {buy_now, wait}`, exactly one `CatalystIdentified` is emitted; if `pass`, no event is emitted.
2. Given 3 distinct insiders post `transaction_type=buy` for ticker `ABC` within 30 days, when the third `InsiderTransactionRecorded` arrives, then `ReevaluateOnInsiderClusterDetected` upserts a `CatalystAssessment(catalyst_type=InsiderClusterBuy)` and emits `CatalystIdentified`.
3. Given only 2 insider buys in 30 days, when `ReevaluateOnInsiderClusterDetected` runs, then no cluster is recorded and no event is emitted.
4. Given a held `AccountPosition` for ticker `DEF` with earnings in 7 days and no active `AlphaCandidate`, when `ScheduledEarningsEntryReview` runs, then a `CatalystAssessment(catalyst_type=EarningsInflection)` is produced and `CatalystIdentified` is emitted (action ≠ `pass`).
5. Given the LLM returns malformed JSON twice, when handled, then `AgentParseError` is logged and no `CatalystIdentified` event is emitted.
6. Given a refresh changes `catalyst_score` from 60 to 62 but `entry_action` stays `wait`, when handled, then no new `CatalystIdentified` event is emitted (no band/action change).
7. Given two passes with identical inputs (`temperature=0`), when both run, then the resulting `entry_action`, `catalyst_score`, and `catalyst_type` are identical.

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Cluster threshold — confirm "3+ insiders in 30 days, same ticker, `transaction_type=buy`" matches framework 11 wording. Should purchase-size (>$100K) also gate cluster eligibility? | Domain expert | Open |
| 2 | Score thresholds for `buy_now` vs `wait` vs `pass`. PRD §4.3 example shows `catalyst_score=78 → buy_now` but no boundaries. | Domain expert | Open |
| 3 | Should each `CatalystType` use its own LLM prompt + schema, or a single prompt that emits a discriminated union? | Domain expert | Open |
| 4 | Earnings-window default — 14 days reasonable? PRD doesn't specify. | Domain expert | Open |
| 5 | When two catalyst types qualify (e.g., insider cluster + earnings inflection), pick the higher score, or compose a multi-type catalyst? Event payload only carries one `catalyst_type`. | Domain expert | Open |
| 6 | Should `pass` outcomes be persisted as `CatalystAssessment` rows for audit, even though no event is emitted? | Domain expert | Open |

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
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0005-llm-agents-return-strict-json.md](../../../architecture/adr/0005-llm-agents-return-strict-json.md), [0004-direct-edgar-fetch-for-insider-and-13f.md](../../../architecture/adr/0004-direct-edgar-fetch-for-insider-and-13f.md)) |
