# Activity Log

## 2026/04/28-1820

**User prompt:** "the remaining 5 — portfolio-construction, rebalancing, execution, monitoring, backtesting." — continue the prior `/ddd` thread to produce Layer 2 use-cases for the last 5 bounded contexts.

**Claude output summary:**
- Read the existing 5 use-cases files + Layer 3 anchors (`context-map.md`, `architecture.md`, `event-catalog.md`, `ubiquitous-language.md`) + the `intent-translation.md` template + PRD/investment-frameworks inputs to lock convention.
- Drafted 5 new `use-cases.md` (`Status: Draft (AI)`) — one per remaining bounded context, all using the same section order, table shapes, and link conventions as the first 5:
  - `portfolio-management/portfolio-construction/use-cases.md` — UC-1..UC-3 (ComposePortfolioTarget / RecomposeOnRegimeChange / ClassifyActionsAgainstCurrentPositions). Encodes PRD §4.5 formula + constraints; explicit `action=close` for held tickers dropped from target (no implicit drops); `Σ target_weight + cash_target == 1`. Hard-fail on `ConstraintConflictError` (no silent clipping).
  - `portfolio-management/rebalancing/use-cases.md` — UC-1..UC-7 covering all 6 PRD §4.6 trigger types + a synchronous compose-and-submit step (anchored to ADR-0003). Defensive-only invariant on regime/insider/risk-alert triggers; per-action `≤ 0.05 × portfolio_value` size cap; no event emission on empty plans.
  - `trade-operations/execution/use-cases.md` — UC-1..UC-6 (Submit / HandleFill / HandleRejection / Cancel / Reconcile / EnforcePaperTradingFlag). Reaffirms context-map rule #1 (sole IBKR caller); idempotency on `(plan_id, action_index)`; IBKR is source-of-truth for `AccountPosition.quantity` on reconciliation; fat-finger band on limit prices.
  - `trade-operations/monitoring/use-cases.md` — UC-1..UC-7 (snapshot + 5 alert paths + resolve). Observational-only; cooldown to suppress noise; market-hours-aware stale-price alerts; counter-idempotent on `(event_name, event_id)`. Open question flagged: do we need an `AlertResolved` event for re-upgrades?
  - `research-validation/backtesting/use-cases.md` — UC-1..UC-5 (Configure / ReplayCadenceTick / SimulateReturns / ComputeMetrics / Compare). Explicit v1 scope: target-weight simulation only — `rebalancing`/`execution` NOT invoked (slippage/fill modelling deferred). Anchored to context-map rule #4 (sandboxed `BacktestEventBus` only — no production-bus traffic). Point-in-time correctness as a hard invariant.
- Cross-checked all `Related Docs` paths from `domains/<ctx>/use-cases.md` (4 levels up to `input/`, 3 levels up to `architecture/`); ADR references match files on disk.
- Cross-checked event names against `event-catalog.md`: every event referenced (PortfolioTargetGenerated, RebalanceTriggered, RebalancePlanProposed, OrderSubmitted/Filled/Rejected, PositionUpdated, PortfolioHealthSnapshotTaken, RiskAlertRaised, BacktestCompleted, LoopOverrun) exists in the catalog with matching publishers/subscribers.
- Surfaced ~40 open questions in total across the 5 files for the domain expert (composition formula constants, drift definition, fat-finger band, threshold tables, cooldown durations, slippage model in backtest, etc.). These block downstream Layer 3 generation per `/ddd` skill rule.

**Todo list status:** all 7 todos completed.

| # | Todo | Status |
|---|------|--------|
| 1 | Review architecture.md + ADRs for context-specific constraints | ✅ |
| 2 | Draft portfolio-construction use-cases.md | ✅ |
| 3 | Draft rebalancing use-cases.md | ✅ |
| 4 | Draft execution use-cases.md | ✅ |
| 5 | Draft monitoring use-cases.md | ✅ |
| 6 | Draft backtesting use-cases.md | ✅ |
| 7 | Update activity-log, commit and push | ✅ |

**Next session:** all 10 Layer 2 use-cases now exist at `Status: Draft (AI)`. Domain-expert review + approval is the gate before Layer 3 (`bounded-context.md`, `aggregate.md`, `domain-event-catalog.md`) can be derived per context. Recommended next: walk one context (suggest `market-data`) end-to-end through Layers 3/4/5 as a reference vertical slice before fanning out.

---

## 2026/04/28-1745

**User prompt:** `continue` — resume the prior `/ddd` thread; user picked option **B** (5 contexts now, then commit/push and stop).

**Claude output summary:**
- Confirmed proposal: generate 5 Layer 2 `use-cases.md` files (`Status: Draft (AI)`) for `market-data`, `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`. All NEW, no overwrites.
- Drafted each file from the `intent-translation.md` template — same sections, same tables. Each carries: Business Intent → Affected Contexts → Term Mapping → Use Cases (UC-N) → Invariants → Cross-Context Deps → Ownership → Constraints → Acceptance Criteria → Open Questions → Related Docs.
- File contents anchored to the existing Layer 3 docs:
  - `market-data/use-cases.md` — UC-1..UC-5 (price/fundamental/Form 4/13F ingest + universe read-model). Reaffirms ADR-0004 EDGAR-direct decision and HR-1 (no cross-context DB reads). Cluster detection delegated to `catalyst-timing`.
  - `alpha-generation/use-cases.md` — UC-1..UC-3 (RunScreeningPass / RescoreOnFundamentalChange / EnableDisableFramework). Anchored to ADR-0005 strict-JSON; composite `AlphaScore` re-normalizes when active framework set changes; PRD §4.1 floors are hard defaults.
  - `conviction-sizing/use-cases.md` — UC-1..UC-2 (Assess / RegradeOnFundamentalChange). Refresh emits only on grade or band change (anti-noise). `portfolio-construction` owns the final cap clip.
  - `catalyst-timing/use-cases.md` — UC-1..UC-3 (Evaluate / ReevaluateOnInsiderClusterDetected / ScheduledEarningsEntryReview). Codifies "no event on `pass`" rule (matches event-catalog payload table). Held-name coverage extends past alpha candidates.
  - `risk-regime/use-cases.md` — UC-1..UC-2 (Recompute / DowngradeOnAlert). Singleton aggregate; `RiskRegimeChanged` emits only on actual transition; alert-driven downgrades are monotonic.
- Cross-check: every `Related Docs` relative path resolves (`../../../../input/...` for PRDs, `../../../architecture/...` for Layer 3, `../../../architecture/adr/...` for ADRs). All ADRs referenced (`0002`, `0004`, `0005`) exist on disk.
- Open questions surfaced per file (vendor selection, grade thresholds, cluster size cap, regime cap table, etc.) — these block downstream Layer 3 generation until a domain expert closes them, per `/ddd` skill rule.

**Todo list status:** all 6 todos completed.

| # | Todo | Status |
|---|------|--------|
| 1 | Write market-data use-cases.md | ✅ |
| 2 | Write alpha-generation use-cases.md | ✅ |
| 3 | Write conviction-sizing use-cases.md | ✅ |
| 4 | Write catalyst-timing use-cases.md | ✅ |
| 5 | Write risk-regime use-cases.md | ✅ |
| 6 | Cross-check links + activity-log + commit/push | ✅ |

**Next session:** generate the remaining 5 Layer 2 `use-cases.md` for `portfolio-construction`, `rebalancing`, `execution`, `monitoring`, `backtesting`. After that, all 10 use-cases sit at `Status: Draft (AI)` awaiting domain-expert approval before Layer 3 generation can be trusted.

---

## 2026/04/28-1720

**User prompt:** `/ddd doc/ddd/input/prd.md doc/ddd/input/investment-frameworks.md` — continue the DDD generation; user chose to start with the 5 ADRs (Layer 6) before the Layer 2 use-cases.

**Claude output summary:**
- Inventoried existing `doc/ddd/output/` (Layer 3 architecture docs + ai-rules already present from the prior session).
- Proposed the full downstream model: 10 use-cases, 10 bounded-contexts, 15 aggregates, 19 integration contracts, 10 OpenAPI + 10 AsyncAPI, 5 ADRs (~89 files total) with a session-by-session execution plan.
- User confirmed scope, picked **ADRs first**, said keep `backtesting` BC modeled, GICS for sector classification, USD only.
- Drafted all 5 ADRs as `Status: Proposed` under `doc/ddd/output/architecture/adr/`:
  - `0001-modular-monolith-hexagonal.md` — modular monolith with per-BC hexagonal style
  - `0002-configuration-shared-kernel.md` — Configuration as VO-only shared kernel (HR-2 anchor)
  - `0003-orchestrator-and-sync-rebalance.md` — Trading Loop orchestrator + sync `rebalancing → execution` command
  - `0004-direct-edgar-fetch-for-insider-and-13f.md` — direct SEC EDGAR fetch with ACL in `market-data`
  - `0005-llm-agents-return-strict-json.md` — LLM agents must use tool-use / structured-output with per-agent JSON Schema
- Reconciliation note: prior activity-log named ADR-0001 "strict-pipeline" and ADR-0004 "ibkr-paper-trading-first", but the Layer 3 docs (`context-map.md`, `architecture.md`) actually cite 0002/0003/0005; ADR-0001 needed to anchor the modular-monolith style, ADR-0004 needed to anchor the EDGAR ACL row. Renumbered titles match what the existing Layer 3 docs already reference.
- Cross-checked Related Docs paths from `architecture/adr/`: relative links resolve correctly. ADR-0003 forward-references `../contracts/rebalancing-execution.md` and ADR-0004 references `../contracts/market-data-catalyst-timing.md` — those files don't exist yet and will be produced in the Layer 4 contract sessions.

**Todo list status:** all 6 todos completed.

| # | Todo | Status |
|---|------|--------|
| 1 | Write ADR-0001 (Modular monolith + hexagonal) | ✅ |
| 2 | Write ADR-0002 (Configuration as Shared Kernel) | ✅ |
| 3 | Write ADR-0003 (Trading Loop + sync command) | ✅ |
| 4 | Write ADR-0004 (Direct SEC EDGAR fetch) | ✅ |
| 5 | Write ADR-0005 (LLM agents return strict JSON) | ✅ |
| 6 | Cross-check ADR links and Related Docs paths; report tree | ✅ |

**Next session:** generate Layer 2 `domains/<ctx>/use-cases.md` for all 10 contexts (`Status: Draft (AI)` — needs domain-expert approval before Layer 3 docs derived from them are trusted). Recommended order: `market-data` → `alpha-generation` → `conviction-sizing` → `catalyst-timing` → `risk-regime` → `portfolio-construction` → `rebalancing` → `execution` → `monitoring` → `backtesting`.

---

## 2026/04/28-1700

**User prompt:** `/ddd C:\projects\fccnc\ai-trader\fccnc_claude_trader\doc\ddd\input` — translate the PRD + investment-frameworks docs into Governed AI Software Delivery documentation (by-domain layout).

**Claude output summary:**
- Read both inputs (`prd.md`, `investment-frameworks.md`) and all `.claude/skills/ddd/templates/*.md`.
- Proposed a 4-domain / 10-bounded-context model and surfaced 14 open questions (PRD conflicts, modelling, scope, session shape).
- After user confirmation (decisions: PRD-wins on conflicts, LLM-driven pluggable agents, central orchestrator, configurable cadence, US-only, single account, USD-only, SEC EDGAR for insider/13F, generate ai-rules in this pass), generated the architecture layer + ai-rules:
  - `doc/ddd/output/architecture/context-map.md`
  - `doc/ddd/output/architecture/architecture.md`
  - `doc/ddd/output/architecture/ubiquitous-language.md`
  - `doc/ddd/output/architecture/event-catalog.md`
  - `doc/ddd/output/ai-rules/development-rules.md`
  - `doc/ddd/output/ai-rules/review-checklist.md`
  - `doc/ddd/output/ai-rules/workflow-templates.md`
- Created `architecture/adr/` and `architecture/contracts/` directory skeletons (empty — populated in later sessions).
- Five ADRs are anticipated and named in the docs (0001 strict-pipeline, 0002 configuration-shared-kernel, 0003 central-orchestrator, 0004 ibkr-paper-trading-first, 0005 llm-driven-agents-strict-json) but not drafted in this pass.

**Todo list status:** all 8 todos completed.

| # | Todo | Status |
|---|------|--------|
| 1 | Generate `architecture/context-map.md` | ✅ |
| 2 | Generate `architecture/architecture.md` | ✅ |
| 3 | Generate `architecture/ubiquitous-language.md` | ✅ |
| 4 | Generate `architecture/event-catalog.md` | ✅ |
| 5 | Generate `ai-rules/development-rules.md` | ✅ |
| 6 | Generate `ai-rules/review-checklist.md` | ✅ |
| 7 | Generate `ai-rules/workflow-templates.md` | ✅ |
| 8 | Cross-check, activity-log, commit & push | ✅ |

**Next session:** generate `domains/<domain>/<ctx>/use-cases.md` per context (Layer 2, `Status: Draft (AI)` — needs domain-expert approval before Layer 3). Recommended order: `signal-intelligence/market-data` → `signal-intelligence/alpha-generation` first.
