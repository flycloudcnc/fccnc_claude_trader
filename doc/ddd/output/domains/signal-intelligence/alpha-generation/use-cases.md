# Intent Translation: signal-intelligence/alpha-generation

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.1 Alpha Screener Agent; §5 Execution Loop), `doc/ddd/input/investment-frameworks.md` (frameworks 1–6, 9, 11, 12)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Run a configurable set of pluggable investment frameworks (Greenblatt, Goldman Small-Cap, Lynch, Mayer, Wood, Camillo, Insider/13F, Blind-Spots) over the current `Universe` and emit a ranked list of `AlphaCandidate`s with a composite `AlphaScore`. Frameworks are LLM-driven Claude prompts returning strict JSON (ADR-0005); each maps to a `Framework` strategy class. This is the entry point of the signal pipeline.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `signal-intelligence/alpha-generation` | Yes | Owns `ScreeningRun`, `AlphaCandidate`, `Framework` plug-ins. |
| `signal-intelligence/market-data` | No | Provides `Universe`, `FundamentalsView`, insider/13F views. |
| `signal-intelligence/conviction-sizing` | No | Reacts to `AlphaCandidateGenerated`. |
| `signal-intelligence/catalyst-timing` | No | Reacts to `AlphaCandidateGenerated`. |
| `portfolio-management/portfolio-construction` | No | Buffers candidates for composition. |
| `research-validation/backtesting` | No | Replays screening over historical windows. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Alpha Screener Agent (PRD §4.1) | AlphaScreenerService (application service over `ScreeningRun`) | `AlphaScreenerService` | Existing |
| Framework / strategy (PRD §11; investment-frameworks.md) | Framework | `Framework` | Existing |
| Candidate (PRD §4.1 output) | AlphaCandidate | `AlphaCandidate` | Existing |
| `alpha_score` (PRD §4.1) | AlphaScore | `AlphaScore` | Existing |
| Filters (market_cap, ROIC, earnings yield, revenue growth, analyst coverage) | (Used as inputs to `Framework`s, not separate VOs) | — | — |
| Insider signal score (PRD §4.1) | Per-framework sub-score (Insider/13F framework) | `framework_scores["insider_13f"]` | Existing |

## Use Cases

### UC-1: RunScreeningPass

- **Actor:** Trading Loop orchestrator (per cadence tick).
- **Trigger:** Orchestrator step "alpha screen" begins for a `RunId`.
- **Goal:** Produce a deterministic, ranked `ScreeningRun` of `AlphaCandidate`s for the current `Universe` by running all enabled `Framework`s.
- **Primary Aggregate:** `ScreeningRun`.
- **Pre-conditions:** `market-data` read-models populated; `Configuration` loaded; ≥ 1 framework enabled; Anthropic API key available.
- **Outcome (success):** `ScreeningRun` persisted with `AlphaCandidate`s (each with composite `AlphaScore` and per-framework sub-scores); `AlphaCandidateGenerated` emitted per candidate.
- **Outcome (failure):** Universe empty → step skipped, log `EmptyUniverseSkipped`. Framework LLM returns malformed JSON after retries → that framework's score is nulled for the pass; `AgentParseError` logged (ADR-0005); `monitoring` alerted. Anthropic outage → entire pass aborts; orchestrator records `LoopOverrun`.
- **Invariants Touched:** `AlphaScore ∈ [0, 100]`; framework sub-scores ∈ [0, 100]; one `AlphaCandidate` per `(screening_run_id, ticker)`; reason ≤ 280 chars (event payload constraint).
- **Cross-context Calls:** Reads `FundamentalsView`, `InsiderClusterView`, `SmartMoneyHoldingsView` from `market-data`; emits `AlphaCandidateGenerated` to `conviction-sizing`, `catalyst-timing`, `portfolio-construction`, `backtesting`.

### UC-2: RescoreCandidateOnFundamentalChange

- **Actor:** `alpha-generation` event subscriber.
- **Trigger:** `FundamentalSnapshotIngested` from `market-data`.
- **Goal:** Re-evaluate eligibility filters for the affected ticker; if still eligible, recompute `AlphaScore` and emit a refreshed `AlphaCandidateGenerated`; if newly ineligible, mark the candidate stale.
- **Primary Aggregate:** `AlphaCandidate` (within the latest open `ScreeningRun`, if any).
- **Pre-conditions:** A `ScreeningRun` exists for the current `RunId`; the affected ticker is in the universe.
- **Outcome (success):** Refreshed candidate persisted with new `event_id`; downstream subscribers dedupe on `(event_name, event_id)`.
- **Outcome (failure):** LLM parse error → candidate kept at last-good score; warning logged. Ticker not in universe → no-op.
- **Invariants Touched:** Reactive update must use the existing `screening_run_id` of the open pass; never split a single pass across two run IDs.
- **Cross-context Calls:** Subscribes to `FundamentalSnapshotIngested`; emits `AlphaCandidateGenerated` (refreshed).

### UC-3: EnableDisableFramework

- **Actor:** Operator (config change) — applied at next process start (Configuration is immutable per ADR-0002).
- **Trigger:** Updated `frameworks.toml` (or equivalent) committed and process restarted.
- **Goal:** Add or remove a `Framework` from the active set without code changes to `AlphaScreenerService`.
- **Primary Aggregate:** `ScreeningRun` (uses the new active framework set on the next pass).
- **Pre-conditions:** Framework prompt + JSON Schema exists under `src/signal_intelligence/alpha_generation/frameworks/`.
- **Outcome (success):** Next `RunScreeningPass` produces sub-scores for the new active set only; disabled frameworks no longer contribute to `AlphaScore`.
- **Outcome (failure):** Missing prompt / schema for a configured framework → process fails fast at startup (loud failure preferred to silent skip).
- **Invariants Touched:** Composite `AlphaScore` formula re-normalizes over the active set so `AlphaScore` remains in `[0, 100]` regardless of how many frameworks are enabled.
- **Cross-context Calls:** None at runtime — config-only.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `AlphaScore ∈ [0, 100]` | `AlphaCandidate` | New |
| One `AlphaCandidate` per `(screening_run_id, ticker)` | `ScreeningRun` | New |
| Composite `AlphaScore` is re-normalized when active framework set changes | `AlphaScreenerService` | New |
| All `Framework` outputs must conform to per-framework JSON Schema (ADR-0005) | `Framework` (Strategy) | New (anchored by ADR-0005) |
| `reason` payload ≤ 280 chars | `AlphaCandidateGenerated` | New (event constraint) |
| One `ScreeningRun` per `RunId` | `ScreeningRun` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `market-data` | Read-model: `FundamentalsView`, `InsiderClusterView`, `SmartMoneyHoldingsView` | Screening inputs. |
| Inbound | `market-data` | Event: `FundamentalSnapshotIngested` | Reactive rescoring. |
| Inbound | `Configuration` (SK) | Shared kernel value objects | `RiskTolerance`, `PortfolioSize`, etc. influence framework selection / weighting. |
| Outbound | `conviction-sizing` | Event: `AlphaCandidateGenerated` | Triggers conviction grading. |
| Outbound | `catalyst-timing` | Event: `AlphaCandidateGenerated` | Triggers catalyst scan. |
| Outbound | `portfolio-construction` | Event: `AlphaCandidateGenerated` | Pass-level composition input. |
| Outbound | `backtesting` | Event: `AlphaCandidateGenerated` (on backtest bus) | Replay verification. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Framework plug-in registry | `alpha-generation` | TBD |
| LLM prompt + JSON Schema per framework | `alpha-generation` | TBD |
| Composite scoring formula + normalization | `alpha-generation` | TBD |
| Eligibility filters from PRD §4.1 | `alpha-generation` | TBD |

## Constraints & Prohibited Actions

1. **Strict JSON only** from every framework LLM agent (ADR-0005). Free-text replies are a hard error; failed parse retries up to 2× then logs `AgentParseError` and zeroes that framework's contribution for the pass.
2. **No direct DB reads** from other contexts (HR-1) — only `market-data` read-models and bus events.
3. **No live order writes** — `alpha-generation` never touches `execution` directly.
4. **Cost discipline** — use Claude Sonnet for high-volume per-ticker screening; reserve Opus for synthesis (per architecture.md). Cache framework prompts via Anthropic prompt cache.
5. **Determinism per `RunId`** — same inputs must produce the same `AlphaScore` (LLM `temperature=0` + cached prompts; record prompt hash in audit log).
6. **PRD §4.1 filters are hard floors** — `market_cap ∈ [100M, 10B]`, `roic_min=15`, `earnings_yield_min=5`, `revenue_growth_min=15`, `analyst_coverage_max=8` are the v1 default; tunable only via `Configuration`.

## Acceptance Criteria (testable)

1. Given a 50-ticker universe with all 8 frameworks enabled, when `RunScreeningPass` completes, then `ScreeningRun.candidates` contains ≤ 50 `AlphaCandidate`s, each with `AlphaScore ∈ [0, 100]` and one entry per active framework in `framework_scores`.
2. Given a framework returns malformed JSON twice, when the second retry parses fail, then `AgentParseError` is logged with prompt + response hashes and that framework contributes `0` to the candidate's composite score for that pass; the pass still completes.
3. Given the active framework set is reduced from 8 to 4, when the next pass runs, then `AlphaScore` for any ticker is computed only over the 4 active frameworks and is still bounded to `[0, 100]`.
4. Given a `FundamentalSnapshotIngested` for a ticker already in the open `ScreeningRun`, when handled, then a refreshed `AlphaCandidateGenerated` is emitted with the **same** `screening_run_id` and a new `event_id`.
5. Given two passes with identical inputs and `temperature=0`, when both run, then `AlphaScore` per ticker is identical (deterministic).
6. Given a ticker fails PRD §4.1 floors after a fundamentals refresh, when the rescoring handler runs, then the candidate is removed from the pass and no `AlphaCandidateGenerated` is emitted for it.

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Confirm v1 active framework set: Greenblatt, Goldman Small-Cap, Lynch, Mayer, Wood, Camillo, Insider/13F, Blind-Spots — all 8 on by default? Pabrai/Marks/Burry are conviction/catalyst-side per PRD; confirm that split. | Domain expert | Open |
| 2 | Composite `AlphaScore` formula — equal-weight average of active framework scores, or weighted (and how)? | Domain expert | Open |
| 3 | Should `AlphaCandidate.reason` be LLM-synthesized from sub-reasons, or framework-specific? Synthesis adds Opus cost. | Domain expert | Open |
| 4 | When is a candidate "stale" vs "removed"? PRD doesn't say. | Domain expert | Open |
| 5 | Should Camillo's social-arbitrage framework run in v1 if no social-data feed exists yet (Phase 2 in PRD §6)? | Domain expert | Open |

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
