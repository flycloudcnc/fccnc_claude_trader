# Intent Translation: signal-intelligence/market-data

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4 agents — data needs; §6 Data Requirements), `doc/ddd/input/investment-frameworks.md` (frameworks 11 Insider/13F)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Provide every downstream agent with normalized, point-in-time market data: prices, fundamentals, insider Form 4 transactions, and 13F-HR filings. Ingestion runs on the Trading-Loop cadence; SEC EDGAR is fetched directly (per session decision #11). This context is the only writer of price/fundamental/insider/13F state and the only adapter to external vendors and EDGAR.

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `signal-intelligence/market-data` | Yes | Owns ingestion + ACL against vendors and SEC EDGAR. |
| `signal-intelligence/alpha-generation` | No | Reads `FundamentalsView`, `SmartMoneyHoldingsView`, `InsiderClusterView`. |
| `signal-intelligence/conviction-sizing` | No | Reads `FundamentalsView`. |
| `signal-intelligence/catalyst-timing` | No | Subscribes to `InsiderTransactionRecorded`. |
| `portfolio-management/risk-regime` | No | Reads breadth/volatility series via `LatestPriceView`. |
| `trade-operations/monitoring` | No | Reads `LatestPriceView` for portfolio valuation. |
| `research-validation/backtesting` | No | Reads time-bounded historical series. |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Price data (PRD §6) | PriceSnapshot | `PriceSnapshot` | Existing |
| Financial statements (PRD §6) | FundamentalSnapshot | `FundamentalSnapshot` | Existing |
| Insider transactions (PRD §6) | InsiderTransaction | `InsiderTransaction` | Existing |
| 13F filings (PRD §6) | ThirteenFFiling | `ThirteenFFiling` | Existing |
| Universe (PRD §4.1 filters) | Universe | `Universe` | Existing |
| Form 4 (SEC) | InsiderTransaction (source = Form 4) | `InsiderTransaction` | Existing |
| 13F-HR (SEC) | ThirteenFFiling (source = 13F-HR) | `ThirteenFFiling` | Existing |

## Use Cases

### UC-1: IngestDailyPriceSnapshot

- **Actor:** Trading Loop orchestrator (scheduled).
- **Trigger:** Cadence tick (intra-day or daily per Configuration).
- **Goal:** Persist a fresh `PriceSnapshot` per ticker in the universe and publish `PriceSnapshotIngested`.
- **Primary Aggregate:** `PriceSnapshot` (one per ticker per `as_of`).
- **Pre-conditions:** Vendor adapter configured; `Universe` resolved for current pass.
- **Outcome (success):** New snapshot rows persisted; `PriceSnapshotIngested` emitted per ticker.
- **Outcome (failure):** Vendor 5xx / timeout — retry with back-off; persistent failure raises `AgentParseError`-equivalent operational log; orchestrator skips downstream consumers that depend on freshness; `monitoring` raises `RiskAlertRaised(severity=warning, category=stale_price_data)`.
- **Invariants Touched:** Snapshot uniqueness on `(ticker, as_of)`; OHLCV monotonicity (high ≥ open/close ≥ low; volume ≥ 0).
- **Cross-context Calls:** Emits `PriceSnapshotIngested` consumed by `monitoring`, `risk-regime`, `backtesting`.

### UC-2: IngestFundamentalSnapshot

- **Actor:** Trading Loop orchestrator (scheduled, lower cadence than prices).
- **Trigger:** Cadence tick OR fundamentals vendor change-feed signal.
- **Goal:** Persist updated `FundamentalSnapshot` with derived ratios (ROIC, earnings yield, revenue growth YoY, margin trend, analyst coverage, market cap) per ticker.
- **Primary Aggregate:** `FundamentalSnapshot`.
- **Pre-conditions:** Vendor adapter configured; ticker present in `Universe` or held in `AccountPositionView`.
- **Outcome (success):** Snapshot persisted; `FundamentalSnapshotIngested` emitted.
- **Outcome (failure):** Vendor failure — retry; persistent failure logs and skips ticker; downstream agents continue with last-known values (subscribers must dedupe on `(event_name, event_id)`).
- **Invariants Touched:** Snapshot uniqueness on `(ticker, period_end, period)`; derived ratios are computed deterministically from raw values (no LLM in market-data).
- **Cross-context Calls:** Emits `FundamentalSnapshotIngested` consumed by `alpha-generation`, `conviction-sizing`.

### UC-3: IngestForm4Filing

- **Actor:** Trading Loop orchestrator (scheduled poll of EDGAR).
- **Trigger:** Cadence tick (e.g., daily EDGAR poll).
- **Goal:** Parse new SEC Form 4 filings into `InsiderTransaction` records and publish per-transaction events.
- **Primary Aggregate:** `InsiderTransaction`.
- **Pre-conditions:** EDGAR ACL configured (per ADR-0004); rate-limit budget available.
- **Outcome (success):** New `InsiderTransaction` rows persisted; `InsiderTransactionRecorded` emitted per transaction.
- **Outcome (failure):** XBRL/SGML parse error — log + dead-letter the offending filing; do not block siblings. EDGAR rate-limit hit — back-off and resume on next pass.
- **Invariants Touched:** "Insider" = SEC Section 16 reporter (officer/director/10%-holder) — see ubiquitous-language; uniqueness on `(filing_accession, transaction_index)`.
- **Cross-context Calls:** Emits `InsiderTransactionRecorded` consumed by `catalyst-timing`, `alpha-generation`. Cluster detection is **downstream** in `catalyst-timing`.

### UC-4: Ingest13FFiling

- **Actor:** Trading Loop orchestrator (scheduled, quarterly).
- **Trigger:** Cadence tick aligned to 13F filing window (45 days post quarter-end).
- **Goal:** Parse new 13F-HR filings for tracked filers (e.g., Berkshire, Third Point) into `ThirteenFFiling` aggregates and publish.
- **Primary Aggregate:** `ThirteenFFiling`.
- **Pre-conditions:** Tracked filer CIK list configured; EDGAR ACL configured.
- **Outcome (success):** Filing persisted with normalized holdings; `ThirteenFFilingRecorded` emitted.
- **Outcome (failure):** Parse error → dead-letter the single filing; rate-limit → back-off.
- **Invariants Touched:** Uniqueness on `(filer_cik, period_end)`; `holdings[].ticker` resolves to known `Ticker` (unresolved tickers logged but not emitted).
- **Cross-context Calls:** Emits `ThirteenFFilingRecorded` consumed by `alpha-generation`.

### UC-5: ServeUniverseSnapshot

- **Actor:** Downstream services (`alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`, `monitoring`, `backtesting`).
- **Trigger:** Sync read-model query at the start of a per-context step.
- **Goal:** Return the universe + latest snapshots applicable for `as_of` (live) or window-bounded for backtests.
- **Primary Aggregate:** `Universe` (read-only projection); backed by underlying snapshots.
- **Pre-conditions:** At least one successful ingestion pass.
- **Outcome (success):** Read-model returns deterministic snapshot for the requested `as_of`.
- **Outcome (failure):** Empty universe → caller skips its step and orchestrator logs `LoopOverrun`-adjacent operational warning; never returns partial-but-mislabeled data.
- **Invariants Touched:** Read-models are point-in-time consistent — no future leakage into backtests.
- **Cross-context Calls:** Sync read-model exposed via `OHS / PL`. No events emitted by this UC directly.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `PriceSnapshot` uniqueness on `(ticker, as_of)` | `PriceSnapshot` / `market-data` | New |
| OHLCV ordering (`low ≤ open,close ≤ high`, `volume ≥ 0`) | `PriceSnapshot` | New |
| `FundamentalSnapshot` uniqueness on `(ticker, period_end, period)` | `FundamentalSnapshot` | New |
| `InsiderTransaction` uniqueness on `(filing_accession, transaction_index)` | `InsiderTransaction` | New |
| Insider role ∈ `{officer, director, 10pct_holder}` (SEC Section 16) | `InsiderTransaction` | New |
| `ThirteenFFiling` uniqueness on `(filer_cik, period_end)` | `ThirteenFFiling` | New |
| Read-models are point-in-time consistent (no future leakage in backtests) | All read-models | New (HR-relevant) |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Outbound | `alpha-generation` | Read-model `FundamentalsView`, `SmartMoneyHoldingsView`, `InsiderClusterView` (raw transactions) | Screening inputs. |
| Outbound | `conviction-sizing` | Read-model `FundamentalsView` | Moat/reinvestment scoring. |
| Outbound | `catalyst-timing` | Event `InsiderTransactionRecorded` + read-model | Cluster detection lives downstream. |
| Outbound | `risk-regime` | Read-model `LatestPriceView` (breadth/volatility series) | Regime computation. |
| Outbound | `monitoring` | Read-model `LatestPriceView` | Portfolio valuation. |
| Outbound | `backtesting` | Sync historical query | Time-bounded replay. |
| Inbound | (External) IEX/Polygon/fundamentals vendor | REST polling (ACL) | Vendor schema → domain VOs. |
| Inbound | (External) SEC EDGAR | HTTP fetch + XBRL/SGML parse (ACL) | Form 4 + 13F-HR. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| Vendor + EDGAR ACL adapters | `market-data` | TBD |
| Snapshot storage and read-models | `market-data` | TBD |
| Cluster detection (downstream) | `catalyst-timing` | TBD |

## Constraints & Prohibited Actions

1. **Direct EDGAR fetch only** for insider/13F (per session decision #11; ADR-0004). No third-party insider-data API in v1.
2. **No LLM calls inside `market-data`** — this context is deterministic ingestion + ACL only. LLM scoring lives in `alpha-generation` / `conviction-sizing` / `catalyst-timing`.
3. **No cross-context DB reads** (HR-1) — downstream contexts must use read-models or events.
4. **Rate-limit aware** — EDGAR requests obey SEC fair-use rules; back-off on 429.
5. **No future leakage** — backtest queries must be `as_of`-bounded.
6. **No live order writes** — `market-data` cannot call IBKR; only `execution` may.

## Acceptance Criteria (testable)

1. Given vendor returns OHLCV for ticker `AAPL` at `2026-04-28T16:00Z`, when ingestion runs, then a `PriceSnapshot(AAPL, 2026-04-28T16:00Z)` row exists exactly once and `PriceSnapshotIngested` is emitted with the same `event_id` even on retry.
2. Given a duplicate `PriceSnapshot` ingest, when the writer commits, then the second write is a no-op and no duplicate event is emitted.
3. Given an EDGAR Form 4 with two sub-transactions, when parsed, then two `InsiderTransaction` rows are persisted with distinct `transaction_index` values and two `InsiderTransactionRecorded` events are emitted.
4. Given an EDGAR rate-limit (429) response, when ingestion runs, then back-off is applied and the operation is retried on the next cadence tick — no exception escapes to the orchestrator.
5. Given a backtest query for `as_of = 2024-06-30`, when read-model serves the universe, then no row with `as_of > 2024-06-30` is returned.
6. Given a 13F-HR for filer `Berkshire`, period_end `2026-03-31`, when ingested twice, then exactly one `ThirteenFFiling` row exists for `(filer_cik, period_end)`.

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Vendor selection (IEX vs Polygon vs other) for prices and fundamentals — which is canonical for v1? | Domain expert | Open |
| 2 | Tracked-filer CIK list for 13F (Berkshire, Third Point, others) — confirm initial set. | Domain expert | Open |
| 3 | Fundamental refresh cadence — daily, weekly, or quarterly only? | Domain expert | Open |
| 4 | EDGAR poll cadence and back-off budget (requests per second). | Domain expert | Open |
| 5 | Universe membership rules — strictly the PRD §4.1 filters, or runtime-tunable? | Domain expert | Open |

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
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) |
