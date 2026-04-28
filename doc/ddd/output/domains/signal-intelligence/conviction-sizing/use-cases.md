# Intent Translation: signal-intelligence/conviction-sizing

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.2 Conviction Agent), `doc/ddd/input/investment-frameworks.md` (frameworks 4 Mayer, 7 Pabrai, 9 Sleep)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

For each `AlphaCandidate`, grade conviction (`A`/`B`/`C`/`D`) and recommend a `TargetWeightRange` using the Pabrai (asymmetric bets), Mayer (100-bagger), and Sleep (scale economies shared) frameworks. Conviction is the sizing input for `portfolio-construction`. LLM-driven, strict-JSON output (ADR-0005).

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `signal-intelligence/conviction-sizing` | Yes | Owns `ConvictionAssessment` aggregate. |
| `signal-intelligence/alpha-generation` | No | Source of `AlphaCandidate`s. |
| `signal-intelligence/market-data` | No | Provides `FundamentalsView`. |
| `portfolio-management/portfolio-construction` | No | Consumes `ConvictionAssessed` for weighting. |
| `portfolio-management/rebalancing` | No | Conviction change is one of the rebalance triggers. |
| `research-validation/backtesting` | No | Replays conviction grading for historical candidates. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Conviction Agent (PRD §4.2) | ConvictionService | `ConvictionService` | Existing |
| Conviction grade (PRD §4.2 output) | ConvictionGrade | `ConvictionGrade` (enum A/B/C/D) | Existing |
| Target weight range (PRD §4.2 output) | TargetWeightRange | `TargetWeightRange` | Existing |
| Downside protection (PRD §4.2 logic) | DownsideProtection | `DownsideProtection` | Existing |
| Upside optionality (PRD §4.2) | UpsideOptionality | `UpsideOptionality` | Existing |
| Moat strength (PRD §4.2) | Moat | `Moat` | Existing |
| Reinvestment ability (PRD §4.2) | (Sub-component of conviction; see UC) | `ReinvestmentScore` | New |
| Management alignment (PRD §4.2) | (Sub-component of conviction) | `ManagementAlignmentScore` | New |

## Use Cases

### UC-1: AssessConvictionForCandidate

- **Actor:** `conviction-sizing` event subscriber.
- **Trigger:** `AlphaCandidateGenerated` from `alpha-generation`.
- **Goal:** Produce a `ConvictionAssessment` for the ticker — a `ConvictionGrade` and `TargetWeightRange` — and emit `ConvictionAssessed`.
- **Primary Aggregate:** `ConvictionAssessment` (one per `(run_id, ticker)`).
- **Pre-conditions:** `FundamentalsView` available for the ticker; Anthropic API reachable.
- **Outcome (success):** Assessment persisted; `ConvictionAssessed` emitted with `conviction_grade`, `target_weight_min`, `target_weight_max`, ≤ 280-char `reason`.
- **Outcome (failure):** Missing fundamentals → grade `D` and emit with reason "insufficient data" (sized at minimum band). LLM parse error after retries → `AgentParseError` logged, candidate deferred to next pass; no event emitted (HR-7 — never emit malformed payload).
- **Invariants Touched:** `0 ≤ target_weight_min ≤ target_weight_max ≤ Configuration.PortfolioSize-implied cap`; `conviction_grade ∈ {A, B, C, D}`; one `ConvictionAssessment` per `(run_id, ticker)`.
- **Cross-context Calls:** Reads `FundamentalsView` from `market-data`; emits `ConvictionAssessed` to `portfolio-construction` and `backtesting`.

### UC-2: RegradeOnFundamentalChange

- **Actor:** `conviction-sizing` event subscriber.
- **Trigger:** `FundamentalSnapshotIngested` for a ticker that already has a `ConvictionAssessment` in the current `RunId`.
- **Goal:** Recompute conviction with updated fundamentals; emit a refreshed `ConvictionAssessed` only if `ConvictionGrade` or `TargetWeightRange` actually changed (suppress noise events).
- **Primary Aggregate:** `ConvictionAssessment`.
- **Pre-conditions:** Existing assessment for `(run_id, ticker)`.
- **Outcome (success):** Assessment updated in place; new event with new `event_id` if grade or band changed; otherwise no-op.
- **Outcome (failure):** LLM error → keep last-good assessment, log warning.
- **Invariants Touched:** Reactive update reuses the same `(run_id, ticker)` key; idempotent on `event_id`.
- **Cross-context Calls:** Subscribes to `FundamentalSnapshotIngested`; emits `ConvictionAssessed` (refreshed, conditional).

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `conviction_grade ∈ {A, B, C, D}` | `ConvictionAssessment` | New |
| `0 ≤ target_weight_min ≤ target_weight_max` | `TargetWeightRange` | New |
| `target_weight_max ≤ Configuration`-derived single-position cap (e.g., `max_single_position` from PRD §4.5; final clip happens in `portfolio-construction`) | `ConvictionAssessment` (advisory) | New |
| One `ConvictionAssessment` per `(run_id, ticker)` | `ConvictionAssessment` | New |
| Event `reason` ≤ 280 chars | `ConvictionAssessed` | New (event constraint) |
| LLM output must conform to `ConvictionAssessment` JSON Schema (ADR-0005) | `ConvictionService` | New |
| Refresh emits only on actual change to grade or band | `ConvictionService` | New (anti-noise) |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `alpha-generation` | Event: `AlphaCandidateGenerated` | Triggers grading. |
| Inbound | `market-data` | Read-model: `FundamentalsView` | Inputs to grading. |
| Inbound | `market-data` | Event: `FundamentalSnapshotIngested` | Reactive regrade. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `RiskTolerance` and `Horizon` shape band selection. |
| Outbound | `portfolio-construction` | Event: `ConvictionAssessed` | Sizing input. |
| Outbound | `rebalancing` | (Indirect — `rebalancing` reacts to `ConvictionChanged` trigger sourced from grade deltas.) | Drift / re-evaluation trigger. |
| Outbound | `backtesting` | Event: `ConvictionAssessed` (on backtest bus) | Replay verification. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Conviction LLM prompt + JSON Schema | `conviction-sizing` | TBD |
| Grade-to-band mapping (A/B/C/D → weight ranges) | `conviction-sizing` | TBD |
| Reactive regrade handler | `conviction-sizing` | TBD |

## Constraints & Prohibited Actions

1. **Strict JSON only** (ADR-0005). LLM must emit a payload matching `ConvictionAssessment` schema; free text → `AgentParseError`.
2. **No direct writes to `portfolio-construction` state** — only via `ConvictionAssessed` events.
3. **No final cap enforcement here** — `conviction-sizing` recommends a band; `portfolio-construction` applies the actual `max_single_position` and regime-driven caps. This BC must not apply portfolio-level constraints (HR-1 / aggregate boundary).
4. **No live order writes**.
5. **Determinism per `RunId`** — same inputs must produce the same grade (LLM `temperature=0`).
6. **No reactive regrade outside the open `RunId`** — once a pass closes, fundamentals updates roll into the next pass.

## Acceptance Criteria (testable)

1. Given an `AlphaCandidateGenerated` for ticker `XYZ` with adequate fundamentals, when `AssessConvictionForCandidate` runs, then a `ConvictionAssessed` event is emitted with `conviction_grade ∈ {A,B,C,D}`, `target_weight_min ≤ target_weight_max`, `reason.length ≤ 280`.
2. Given the same `AlphaCandidate` is delivered twice (at-least-once bus), when handled, then exactly one `ConvictionAssessment` row exists for `(run_id, ticker)` and only one `ConvictionAssessed` event with a stable `event_id` is observable to downstream subscribers (idempotency on `(event_name, event_id)`).
3. Given fundamentals refresh changes ROIC materially but does not flip the grade or band, when `RegradeOnFundamentalChange` runs, then no new `ConvictionAssessed` event is emitted.
4. Given fundamentals refresh flips the grade from `B` to `C`, when handled, then a new `ConvictionAssessed` with a new `event_id` is emitted carrying the same `run_id`.
5. Given the LLM returns malformed JSON twice, when handled, then no event is emitted, `AgentParseError` is logged with prompt + response hashes, and the candidate is deferred (no half-baked `ConvictionAssessed` payload reaches the bus).
6. Given two passes with identical inputs (`temperature=0`), when both run, then the resulting `conviction_grade` and `target_weight_range` are identical.

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Numerical thresholds mapping `A`/`B`/`C`/`D`. PRD §4.2 only shows example output `B → [0.03, 0.06]`. Need full table. | Domain expert | Open |
| 2 | Reinvestment + management-alignment scoring rubric — Mayer/Sleep frameworks suggest qualitative checks; what concrete inputs should the LLM receive? | Domain expert | Open |
| 3 | Should `Horizon` (e.g., 6–24 months) influence which grade-to-band mapping is used? | Domain expert | Open |
| 4 | If `FundamentalsView` is missing entirely for a ticker, is `D` (low) the correct fallback, or should the ticker be dropped (no event)? | Domain expert | Open |
| 5 | Should grade deltas (e.g., `B → A`) emit a separate `ConvictionUpgraded` / `ConvictionDowngraded` event for `rebalancing`, or is `ConvictionAssessed` enough? | Domain expert | Open |

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
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0005-llm-agents-return-strict-json.md](../../../architecture/adr/0005-llm-agents-return-strict-json.md)) |
