Here is the **merged, implementation-ready `prd.md`** for your IBKR portfolio auto-management system, combining architecture + agent framework mapping into one coherent document for a Claude Code agent.

---

# 📄 PRD: AI Portfolio Auto-Management System (IBKR)

## 1. Overview

### 1.1 Product Name

**Fccnc Auto Portfolio Manager (FAPM)**

### 1.2 Objective

Build an AI-driven system that:

* Generates investment ideas (alpha)
* Constructs a risk-aware portfolio
* Executes trades via Interactive Brokers (IBKR)
* Continuously monitors and rebalances positions

---

## 2. Core Design Principle

> **Strict separation of responsibilities**

```text
Alpha (what to buy)
→ Conviction (how much)
→ Risk (how safe)
→ Portfolio (allocation)
→ Execution (orders)
→ Monitoring (feedback loop)
```

---

## 3. System Architecture

```text
Market + Fundamental + Insider Data
        ↓
Alpha Screener Agent
        ↓
Conviction Agent
        ↓
Catalyst Timing Agent
        ↓
Risk Regime Agent
        ↓
Portfolio Construction Agent
        ↓
Rebalancing Agent
        ↓
Execution Agent (IBKR)
        ↓
Monitoring Agent
```

---

## 4. Agent Specifications

---

# 4.1 Alpha Screener Agent

### Frameworks Used

* Greenblatt (Magic Formula)
* Goldman Small-Cap Hidden Gems
* Insider Buying / 13F

### Purpose

Generate investable candidates.

### Logic

```text
alpha_score =
  ROIC score
  + earnings yield score
  + revenue growth score
  + margin trend score
  + insider signal score
```

### Filters

```json
{
  "market_cap": "100M-10B",
  "roic_min": 15,
  "earnings_yield_min": 5,
  "revenue_growth_min": 15,
  "analyst_coverage_max": 8
}
```

### Output

```json
{
  "candidates": [
    {
      "ticker": "XYZ",
      "alpha_score": 82,
      "reason": "High ROIC + improving margins + low coverage"
    }
  ]
}
```

---

# 4.2 Conviction Agent

### Frameworks Used

* Pabrai (asymmetric bets)
* Mayer (100-bagger)
* Nick Sleep (scale economies shared)

### Purpose

Determine position size potential.

### Logic

```text
conviction =
  downside protection
  + upside optionality
  + moat strength
  + reinvestment ability
  + management alignment
```

### Output

```json
{
  "ticker": "XYZ",
  "conviction_grade": "B",
  "target_weight_range": [0.03, 0.06],
  "reason": "Strong reinvestment + asymmetric upside"
}
```

---

# 4.3 Catalyst Timing Agent

### Frameworks Used

* Michael Burry (catalysts)
* Howard Marks (mispricing)
* Insider Buying

### Purpose

Decide entry timing.

### Logic

```text
buy = mispricing + catalyst + market misunderstanding
```

### Catalyst Types

* Earnings inflection
* Spinoff
* Buyback
* Insider buying cluster
* Regulatory change
* Margin recovery

### Output

```json
{
  "ticker": "XYZ",
  "action": "buy_now",
  "catalyst_score": 78,
  "timeframe": "3-9 months"
}
```

---

# 4.4 Risk Regime Agent

### Frameworks Used

* Howard Marks (market cycles)
* Pabrai (margin of safety)
* Burry (downside awareness)

### Purpose

Control total portfolio exposure.

### Regimes

```text
risk_on
neutral
risk_off
capital_preservation
```

### Output

```json
{
  "risk_regime": "neutral",
  "max_equity_exposure": 0.75,
  "cash_target": 0.15,
  "max_position": 0.08
}
```

---

# 4.5 Portfolio Construction Agent

### Frameworks Used

* Pabrai (position sizing)
* Greenblatt (ranking)
* Marks (risk control)

### Purpose

Convert signals into weights.

### Formula

```text
target_weight =
  alpha_score
  × conviction_score
  × catalyst_score
  × risk_adjustment
```

### Constraints

```json
{
  "max_positions": 20,
  "max_single_position": 0.10,
  "max_sector": 0.25,
  "cash_min": 0.05,
  "cash_max": 0.50
}
```

### Output

```json
{
  "portfolio_targets": [
    {
      "ticker": "XYZ",
      "target_weight": 0.06,
      "action": "increase"
    }
  ]
}
```

---

# 4.6 Rebalancing Agent

### Frameworks Used

* Marks (second-level thinking)
* Burry (catalyst timeline)
* Pabrai (re-evaluation)

### Purpose

Trigger portfolio adjustments.

### Triggers

```json
[
  "weight_drift > 25%",
  "conviction_change",
  "risk_regime_change",
  "price_move > 20%",
  "earnings_event",
  "insider_selling"
]
```

### Output

```json
{
  "rebalance": true,
  "action": "trim",
  "reason": "Position exceeded target after rally"
}
```

---

# 4.7 Execution Agent

### Purpose

Execute trades via IBKR safely.

### Responsibilities

```text
1. Generate orders
2. Validate constraints
3. Submit to IBKR API
4. Confirm fills
5. Update portfolio state
```

### Order Policy

```json
{
  "default_order": "limit",
  "max_order_size": 0.05,
  "paper_trading": true
}
```

---

# 4.8 Monitoring Agent

### Frameworks Used

* Marks (risk awareness)
* Pabrai (downside protection)

### Purpose

Ensure portfolio safety.

### Metrics

```json
{
  "drawdown": "",
  "volatility": "",
  "sector_exposure": "",
  "cash": "",
  "benchmark_return": ""
}
```

### Output

```json
{
  "risk_status": "warning",
  "issue": "High sector concentration",
  "action": "reduce exposure"
}
```

---

## 5. Execution Loop

```python
while market_open:

    signals = alpha_screener()
    conviction = conviction_agent(signals)
    timing = catalyst_agent(signals)
    regime = risk_regime_agent()

    portfolio = construct_portfolio(signals, conviction, timing, regime)
    rebalance = check_rebalance(portfolio)

    if rebalance:
        orders = generate_orders(portfolio)
        execute_orders_ibkr(orders)

    monitor_portfolio()

    sleep(interval)
```

---

## 6. Data Requirements

### Required

* Price data
* Financial statements
* Insider transactions
* 13F filings

### Optional (Phase 2)

* Social sentiment
* News NLP
* alternative data

---

## 7. Configuration (User Input)

```json
{
  "risk_tolerance": "medium",
  "target_return": 0.15,
  "max_drawdown": 0.15,
  "horizon": "6-24 months",
  "portfolio_size": 20
}
```

---

## 8. MVP Scope

### Phase 1

* Alpha Screener
* Conviction Agent
* Portfolio Construction
* IBKR Paper Trading
* Monitoring

### Phase 2

* Risk Regime Agent
* Rebalancing Agent
* Catalyst Timing Agent

### Phase 3

* Insider + Social Signals
* Advanced execution (VWAP)
* Backtesting system

---

## 9. Critical Risks

### 1. False Alpha

Signals may not translate into returns.

### 2. Execution Risk

IBKR API errors, slippage.

### 3. Correlation Risk

Portfolio appears diversified but isn’t.

### 4. Regime Shift

Strategies fail in new market conditions.

---

## 10. Key Insight

> This system will succeed or fail based on **discipline and risk control**, not idea generation.

---

## 11. Implementation Notes for Claude Code

### Requirements

* Strict JSON outputs (no free text)
* Deterministic agent interfaces
* Modular agent architecture
* Parallel execution where possible
* Tool-first design (data APIs)

---

## 12. Next Step (Recommended)

To move from PRD → working system:

1. Build **IBKR execution service (paper trading first)**
2. Implement **Alpha + Portfolio + Risk loop**
3. Add **backtesting harness**
4. Then layer additional signals

---

