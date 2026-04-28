# Ubiquitous Language Glossary

**Version:** 0.1
**Last Updated:** 2026-04-28

## System-Wide Terms

| Term | Definition | Code Name | Example |
|------|-----------|-----------|---------|
| Ticker | Unique exchange symbol identifying a tradeable security (US-listed equity in v1). | `Ticker` | `Ticker("AAPL")` |
| Money | Monetary amount with currency (USD only in v1, decision #12). | `Money` | `Money(1234.50, "USD")` |
| Weight | Fractional portfolio weight in `[0, 1]`. | `Weight` | `Weight(0.06)` for 6%. |
| Percent | Fractional value in `[0, 1]` used outside portfolio weights (e.g., scores, ratios). | `Percent` | `Percent(0.15)` for 15%. |
| Universe | The set of tickers eligible for screening at the current pass. | `Universe` | US equities, market-cap $100M–$10B (PRD §4.1). |
| Run | One end-to-end pass of the Trading Loop, identified by a `RunId` correlating all events of that pass. | `RunId`, `Run` | `RunId("2026-04-28T0930-001")` |
| Configuration | Immutable user-supplied portfolio settings: `RiskTolerance`, `TargetReturn`, `MaxDrawdown`, `Horizon`, `PortfolioSize`. Loaded once at process start. | `Configuration` | See ADR-0002. |
| RiskTolerance | User-declared risk appetite — `low`, `medium`, `high`. | `RiskTolerance` (enum) | `RiskTolerance.MEDIUM` |
| TargetReturn | Annualized return goal as a fraction (e.g., 0.15 = 15%). | `TargetReturn` | `TargetReturn(0.15)` |
| MaxDrawdown | Maximum tolerated peak-to-trough loss as a fraction. | `MaxDrawdown` | `MaxDrawdown(0.15)` |
| Horizon | Investment horizon in months (range or single value). | `Horizon` | `Horizon(min_months=6, max_months=24)` |
| PortfolioSize | Target number of positions in the portfolio. | `PortfolioSize` | `PortfolioSize(20)` |
| AlphaScore | Composite 0–100 score from `alpha-generation` quantifying expected outperformance. | `AlphaScore` | `AlphaScore(82)` — PRD §4.1 example. |
| ConvictionGrade | Letter grade `A`/`B`/`C`/`D` from `conviction-sizing` indicating sizing confidence. | `ConvictionGrade` (enum) | `ConvictionGrade.B` |
| TargetWeightRange | `(min, max)` weight range recommended by `conviction-sizing`. | `TargetWeightRange` | `TargetWeightRange(0.03, 0.06)` |
| CatalystScore | 0–100 score from `catalyst-timing` for the strength of a near-term catalyst. | `CatalystScore` | `CatalystScore(78)` |
| RiskRegime | Portfolio-wide regime: `risk_on`, `neutral`, `risk_off`, `capital_preservation`. | `RiskRegime` (enum) | `RiskRegime.NEUTRAL` |
| PortfolioTarget | The set of `TargetPosition`s the portfolio should hold after the current pass. | `PortfolioTarget` | Aggregate root in `portfolio-construction`. |
| TargetPosition | One ticker's target weight + action (`open`, `increase`, `trim`, `close`, `hold`) within a `PortfolioTarget`. | `TargetPosition` | Entity inside `PortfolioTarget`. |
| RebalancePlan | Concrete set of `RebalanceAction`s computed by `rebalancing` to move from current positions to a `PortfolioTarget`. | `RebalancePlan` | Aggregate root in `rebalancing`. |
| Order | An instruction submitted to IBKR (limit by default per PRD §4.7); root of the `Order` aggregate in `execution`. | `Order` | `Order(side=BUY, ticker=AAPL, qty=100, type=LIMIT, limit=180.00)` |
| Fill | A reported execution of part or all of an `Order`. | `Fill` | Entity inside `Order`. |
| AccountPosition | The current quantity + cost basis the account holds for a given ticker. | `AccountPosition` | Aggregate root in `execution`. |
| PortfolioHealthSnapshot | Periodic snapshot of `drawdown`, `volatility`, `sector_exposure`, `cash`, `benchmark_return` (PRD §4.8). | `PortfolioHealthSnapshot` | Aggregate root in `monitoring`. |
| RiskAlert | A finding from `monitoring` requiring attention (e.g., `HighSectorConcentration`). | `RiskAlert` | Aggregate root in `monitoring`. |
| Backtest | A historical replay of the pipeline over a fixed window with a fixed strategy configuration. | `Backtest` | Aggregate root in `backtesting`. |
| TradingLoop | The orchestrator application service that drives one `Run` through every context per cadence. | `TradingLoop` | Not an aggregate — application service (see `architecture.md`). |

## Context-Specific Terms

### `signal-intelligence/market-data`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| PriceSnapshot | Daily OHLCV record for one ticker on one date. | `PriceSnapshot` | — |
| FundamentalSnapshot | Point-in-time financial statement values + derived ratios (ROIC, earnings yield, revenue growth, margin trend). | `FundamentalSnapshot` | — |
| InsiderTransaction | One reported Form 4 transaction (buy/sell, qty, price, date, insider role). | `InsiderTransaction` | "Insider" here is an SEC-defined Section 16 officer/director/10%-holder, not just any employee. |
| ThirteenFFiling | One quarterly 13F-HR filing parsed from EDGAR; aggregates positions a fund manager held at quarter-end. | `ThirteenFFiling` | — |
| Cluster | (Detected downstream in `catalyst-timing`) — defined here as a candidate set of `InsiderTransaction`s grouped by ticker + 30-day window. | `InsiderCluster` | The catalyst score for a cluster is computed in `catalyst-timing`; `market-data` only emits the raw transactions. |

### `signal-intelligence/alpha-generation`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| Framework | A pluggable scoring strategy (Greenblatt / Goldman / Lynch / Mayer / Wood / Camillo / Insider+13F / Blind-Spots). | `Framework` (Strategy interface) | A "framework" elsewhere may mean a software framework — here it always means an investment screen. |
| ScreeningRun | One full pass of all enabled frameworks over the universe, producing a ranked list of `AlphaCandidate`s. | `ScreeningRun` | Aggregate root. |
| AlphaCandidate | A ticker that passed at least one framework's filters, with a composite `AlphaScore` and per-framework sub-scores. | `AlphaCandidate` | Entity inside `ScreeningRun`. |

### `signal-intelligence/conviction-sizing`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| ConvictionAssessment | The output of grading one `AlphaCandidate` for sizing potential. | `ConvictionAssessment` | Aggregate root. |
| DownsideProtection | Sub-component of conviction: estimate of capital preservation if the thesis is wrong. | `DownsideProtection` | — |
| UpsideOptionality | Sub-component of conviction: asymmetric upside potential (Pabrai). | `UpsideOptionality` | — |
| Moat | Sub-component of conviction: durability of competitive advantage. | `Moat` | "Moat" in `catalyst-timing` is not used; it lives only here. |

### `signal-intelligence/catalyst-timing`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| CatalystAssessment | Output of evaluating an `AlphaCandidate` for a near-term catalyst. | `CatalystAssessment` | Aggregate root. |
| CatalystType | Enum: `EarningsInflection`, `Spinoff`, `Buyback`, `InsiderClusterBuy`, `RegulatoryChange`, `MarginRecovery`. | `CatalystType` | Mirrors PRD §4.3. |
| EntryAction | Recommended timing action: `buy_now`, `wait`, `pass`. | `EntryAction` | Distinct from `RebalanceAction` (which is portfolio-level). |
| Mispricing | Gap between intrinsic value estimate and market price. | `Mispricing` | — |

### `portfolio-management/risk-regime`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| RiskRegimeState | Current regime + caps it implies (`max_equity_exposure`, `cash_target`, `max_position`). | `RiskRegimeState` | Aggregate root. |

### `portfolio-management/portfolio-construction`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| Constraint | A hard rule the target must satisfy: `max_positions`, `max_single_position`, `max_sector`, `cash_min`, `cash_max` (PRD §4.5). | `Constraint` | — |
| RiskAdjustment | Multiplier derived from the current `RiskRegime` applied to candidate weights. | `RiskAdjustment` | — |

### `portfolio-management/rebalancing`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| RebalanceTrigger | One of: `WeightDriftExceeded`, `ConvictionChanged`, `RiskRegimeChanged`, `PriceMoveExceeded`, `EarningsEvent`, `InsiderSelling` (PRD §4.6). | `RebalanceTrigger` | — |
| RebalanceAction | A single trim/add/open/close instruction on one ticker. | `RebalanceAction` | Distinct from `EntryAction` (catalyst-timing). |
| WeightDrift | Absolute difference between current and target weight as a fraction of target. | `WeightDrift` | — |

### `trade-operations/execution`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| OrderType | `LIMIT` (default per PRD §4.7), `MARKET`, `STOP`, `STOP_LIMIT`. | `OrderType` | — |
| OrderStatus | `PENDING_SUBMIT`, `SUBMITTED`, `PARTIALLY_FILLED`, `FILLED`, `CANCELLED`, `REJECTED`. | `OrderStatus` | Mirrors IBKR's status set, ACL-translated. |
| PaperTradingFlag | When `true`, orders are sent to IBKR paper account; when `false`, live account (ADR-0004). | `paper_trading: bool` | — |

### `trade-operations/monitoring`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| Drawdown | Peak-to-trough loss of the portfolio over the lookback window. | `Drawdown` | — |
| SectorExposure | Fraction of portfolio value invested in each GICS sector. | `SectorExposure` | — |
| AlertSeverity | `info`, `warning`, `critical`. | `AlertSeverity` | — |

### `research-validation/backtesting`

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| BacktestWindow | Inclusive `[start_date, end_date]` historical range. | `BacktestWindow` | — |
| BacktestStrategy | The full configuration set (enabled frameworks, constraints, regime model) under test. | `BacktestStrategy` | — |
| BacktestRun | One execution of the pipeline against a `BacktestWindow` with a `BacktestStrategy`. | `BacktestRun` | Aggregate root. |

## Synonyms and Aliases

| Preferred Term | Aliases (Do Not Use) | Reason |
|---------------|---------------------|--------|
| `Ticker` | `Symbol`, `Stock`, `Instrument` | One canonical name; `Symbol` collides with Python `symbol` type. |
| `AlphaCandidate` | `Pick`, `Idea`, `Suggestion` | Pipeline-stage-specific name. |
| `PortfolioTarget` | `Allocation`, `Plan`, `BookOfBusiness` | "Plan" is reserved for `RebalancePlan`. |
| `RebalancePlan` | `Trade list`, `OrderList` | "Order" is the execution-context aggregate; the plan is upstream of orders. |
| `Order` | `Trade`, `Transaction` | "Trade" is ambiguous (could mean `Fill`). |
| `Fill` | `Execution`, `Trade` | "Execution" is the bounded-context name. |
| `RiskAlert` | `Warning`, `Notification` | "Warning" is a severity level, not a type. |
| `RunId` | `JobId`, `PassId`, `CycleId` | One canonical pass identifier. |
| `Framework` | `Strategy`, `Screener`, `Model` | "Strategy" is the GoF pattern term used at the code level, but the ubiquitous-language term is `Framework`. |

## Naming Conventions

| Category | Convention | Example |
|----------|-----------|---------|
| **Aggregates / Entities** | PascalCase noun | `Order`, `AccountPosition`, `RebalancePlan` |
| **Value Objects** | PascalCase noun/adjective | `Money`, `Weight`, `RiskTolerance`, `AlphaScore` |
| **Events** | PascalCase past tense | `OrderFilled`, `AlphaCandidateGenerated`, `RiskRegimeChanged` |
| **Commands** | PascalCase imperative | `SubmitRebalancePlan`, `RunScreening`, `IngestPriceSnapshot` |
| **Services** | PascalCase noun + Service | `AlphaScreenerService`, `ExecutionService` |
| **Repositories** | PascalCase noun + Repository | `OrderRepository`, `PortfolioTargetRepository` |
| **Frameworks (alpha)** | PascalCase + `Framework` suffix | `GreenblattFramework`, `GoldmanSmallCapFramework` |
| **Adapters (infra)** | PascalCase + `Client`/`Adapter` | `IbkrClient`, `EdgarClient` |

## Ambiguous Terms to Avoid

| Term | Problem | Use Instead |
|------|---------|-------------|
| "Position" | Could mean: target weight (`TargetPosition`), held shares (`AccountPosition`), or trade direction (long/short). | Use the qualified term: `TargetPosition` (construction), `AccountPosition` (execution), or `OrderSide` (long/short). |
| "Signal" | Used loosely across alpha / conviction / catalyst contexts. | Use the specific event: `AlphaCandidateGenerated`, `ConvictionAssessed`, `CatalystIdentified`. |
| "Risk" | Means a regime in one place, an alert in another, a constraint elsewhere. | Use `RiskRegime`, `RiskAlert`, or `Constraint`. |
| "Score" | Each context has its own. | Use `AlphaScore`, `ConvictionGrade`, `CatalystScore`. |
| "Insider" | SEC term (Section 16 reporter) vs. casual "company employee". | Always SEC definition in this codebase. |
| "Rebalance" as a verb | Confused with "trade" or "execute". | Use `RebalancePlan` (the noun output) or `RebalanceService.evaluate(...)`. |

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md](context-map.md) |
| Architecture | [architecture.md](architecture.md) |
| Event Catalog | [event-catalog.md](event-catalog.md) |
| Per-context overrides | `domains/<ctx>/ubiquitous-language.md` (created in later sessions, override-only) |
