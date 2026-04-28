# Intent Translation: trade-operations/execution

**Version:** 0.1
**Last Updated:** 2026-04-28
**Source PRDs / Epics:** `doc/ddd/input/prd.md` (§4.7 Execution Agent; §5 Execution Loop; §11 Implementation Notes)
**Domain Expert (validator):** TBD
**Status:** Draft (AI)

## Business Intent Summary

Receive a `RebalancePlan` from `rebalancing` (synchronous command per ADR-0003), translate each `RebalanceAction` into one or more `Order`s (default `LIMIT`, paper-trading first per PRD §4.7), validate pre-trade constraints, submit to the IBKR adapter, then track fills, rejections, and cancellations until each order reaches a terminal state. Maintain authoritative `AccountPosition` state from IBKR-confirmed fills and reconciliation. Publish `OrderSubmitted`, `OrderFilled`, `OrderRejected`, `PositionUpdated` for `monitoring` and the next-pass `portfolio-construction` read-model. **This is the only context permitted to call IBKR** (context-map dependency rule #1).

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| `trade-operations/execution` | Yes | Owns `Order` and `AccountPosition` aggregates; owns IBKR adapter. |
| `portfolio-management/rebalancing` | No | Source of `SubmitRebalancePlan` sync command; consumes `OrderRejected`. |
| `trade-operations/monitoring` | No | Subscribes to `OrderSubmitted`, `OrderFilled`, `OrderRejected`, `PositionUpdated` for health metrics + alerts. |
| `portfolio-management/portfolio-construction` | No | Reads `AccountPositionView` for action classification. |
| `signal-intelligence/catalyst-timing` | No | Reads `AccountPositionView` for held-name catalyst coverage. |
| `signal-intelligence/market-data` | No | Provides `LatestPriceView` for limit-price floors / sanity checks (read-only). |

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| Execution Agent (PRD §4.7) | ExecutionService | `ExecutionService` | Existing |
| Order (PRD §4.7) | Order aggregate | `Order` | Existing |
| Fill (PRD §4.7 "Confirm fills") | Fill entity | `Fill` | Existing |
| Order policy (PRD §4.7: `default_order=limit`, `max_order_size=0.05`, `paper_trading=true`) | OrderType + per-action size cap + PaperTradingFlag | `OrderType.LIMIT`, `paper_trading: bool` | Existing |
| Portfolio state (PRD §4.7: "Update portfolio state") | AccountPosition aggregate | `AccountPosition` | Existing |
| IBKR API | IBKR adapter (ACL) | `IbkrClient` | Existing |
| Order status (IBKR field) | OrderStatus enum (ACL-translated) | `OrderStatus ∈ {PENDING_SUBMIT, SUBMITTED, PARTIALLY_FILLED, FILLED, CANCELLED, REJECTED}` | Existing |

## Use Cases

### UC-1: SubmitRebalancePlan

- **Actor:** `RebalanceService` (sync caller from `rebalancing`).
- **Trigger:** Sync command `SubmitRebalancePlan(plan_id)` from `rebalancing` per ADR-0003.
- **Goal:** For each `RebalanceAction` in the plan, perform pre-trade validation, create one `Order` aggregate, submit to IBKR, return per-action result (`submitted` / `rejected`) synchronously to the caller. Emit `OrderSubmitted` per accepted order.
- **Primary Aggregate:** `Order` (one per `RebalanceAction`).
- **Pre-conditions:** `RebalancePlan` is loadable by `plan_id`; IBKR session is connected (or reconnect attempted with bounded back-off); `LatestPriceView` available for sanity-check.
- **Outcome (success):** Each accepted action → `Order(status=SUBMITTED)` persisted, `OrderSubmitted` emitted, IBKR `order_id` captured. Sync return is `{submitted: [...], rejected: [...]}` with per-action detail.
- **Outcome (failure):** Pre-trade validation failure → `OrderRejected(reason_code=PRE_TRADE_VALIDATION, reason="…")` emitted; that action does not reach IBKR. IBKR connection lost → bounded reconnect; if not recovered within timeout (default 60s per architecture.md cross-cutting), entire command returns failure to caller; orchestrator decides whether to halt loop. IBKR rejects on submit → `OrderRejected(reason_code=<IBKR code>)`; sync result includes the rejection.
- **Invariants Touched:** Per-`Order`: `qty > 0`; `side ∈ {BUY, SELL}`; if `order_type == LIMIT` then `limit_price > 0`; sanity-check `|limit_price − last_price| / last_price ≤ configured fat-finger band` (e.g., 5%). Idempotency: `(plan_id, action_index)` uniqueness — duplicate `SubmitRebalancePlan(plan_id)` MUST NOT submit duplicate orders to IBKR. `paper_trading` flag flows from `Configuration` (ADR-0004) and is recorded on every `Order` for audit.
- **Cross-context Calls:** Sync inbound from `rebalancing`; emits `OrderSubmitted` per accepted order, `OrderRejected` per rejected action; calls IBKR adapter (only context allowed to do so per context-map dependency rule).

### UC-2: HandleFillFromIbkr

- **Actor:** IBKR adapter callback (ACL inbound).
- **Trigger:** IBKR fill notification (`execDetails` or equivalent) for an open `Order`.
- **Goal:** Append a `Fill` to the `Order`, update `OrderStatus` (`PARTIALLY_FILLED` or `FILLED`), update the corresponding `AccountPosition` (`quantity`, `avg_cost`), emit `OrderFilled` and `PositionUpdated`.
- **Primary Aggregate:** `Order` (and `AccountPosition` as a second aggregate updated atomically per HR-1: each in its own repository, but within the same handler unit-of-work).
- **Pre-conditions:** Open `Order` exists for the IBKR `order_id`; fill `qty > 0` and `qty ≤ remaining order qty`.
- **Outcome (success):** `Fill` persisted; if cumulative filled = ordered qty → `OrderStatus=FILLED` and `OrderFilled(is_complete=true)`; else `OrderStatus=PARTIALLY_FILLED` and `OrderFilled(is_complete=false)`. `AccountPosition` `quantity` and `avg_cost` updated; `PositionUpdated` emitted.
- **Outcome (failure):** Unknown IBKR `order_id` (e.g., out-of-band trade) → log `IbkrOrphanFill` and run reconciliation (UC-5) before raising any event. Duplicate fill notification (same `exec_id`) → de-duplicated; no change.
- **Invariants Touched:** `Σ fills.qty ≤ Order.qty` always; `AccountPosition.quantity` is signed (positive long, negative short — though v1 long-only per PRD); `avg_cost` recomputed as weighted average on BUY fills, unchanged on SELL fills (FIFO accounting reserved for tax — out of scope v1).
- **Cross-context Calls:** Emits `OrderFilled` and `PositionUpdated` to `monitoring` (and read by `portfolio-construction` next pass via `AccountPositionView`).

### UC-3: HandleRejectionFromIbkr

- **Actor:** IBKR adapter callback.
- **Trigger:** IBKR rejection (`error` callback with order id) for a submitted `Order`.
- **Goal:** Update `OrderStatus=REJECTED`, capture `reason_code` from IBKR (ACL-translated to internal codes), emit `OrderRejected`.
- **Primary Aggregate:** `Order`.
- **Pre-conditions:** Open `Order` exists for the IBKR `order_id`.
- **Outcome (success):** `Order(status=REJECTED)` persisted; `OrderRejected` emitted with `reason_code` and human-readable `reason`. `rebalancing` consumes the event and decides retry policy (UC-7 in `rebalancing/use-cases.md`).
- **Outcome (failure):** Unknown IBKR `order_id` → log; raise alert via `monitoring` for investigation.
- **Invariants Touched:** Once `REJECTED`, an `Order` is terminal — no further fills or status changes allowed; any subsequent IBKR callback for that id is logged as anomaly.
- **Cross-context Calls:** Emits `OrderRejected` to `monitoring` and `rebalancing`.

### UC-4: CancelOpenOrder

- **Actor:** Internal — invoked by `ExecutionService` during shutdown, end-of-cadence cleanup, or on configured `time_in_force` expiry. Optionally exposed as a sync command from `rebalancing` for explicit cancel.
- **Trigger:** Cadence end (orchestrator hook), shutdown hook, or `cancel_order(order_id)` sync call.
- **Goal:** Send IBKR cancel for the `Order`; on confirmation set `OrderStatus=CANCELLED`; do not emit a domain event in v1 (cancellation is operational; the next pass's `AccountPositionView` reflects the unchanged state). Open question: do we need a `OrderCancelled` event for `monitoring`?
- **Primary Aggregate:** `Order`.
- **Pre-conditions:** `Order.status ∈ {PENDING_SUBMIT, SUBMITTED, PARTIALLY_FILLED}`.
- **Outcome (success):** IBKR confirms cancel → `OrderStatus=CANCELLED`; partial fills already received remain in `AccountPosition` (already accounted for via UC-2).
- **Outcome (failure):** IBKR cancel fails (already filled in race) → status converges to `FILLED` via UC-2; cancel request becomes a no-op.
- **Invariants Touched:** Cancel is idempotent — multiple cancel requests for the same `order_id` collapse to one IBKR call. Once `CANCELLED` or `FILLED`, an `Order` is terminal.
- **Cross-context Calls:** Calls IBKR adapter; no domain event in v1 (subject to open question #4).

### UC-5: ReconcileWithIbkrAccount

- **Actor:** Trading Loop orchestrator (cadence start) + on `IbkrConnectionLost` recovery.
- **Trigger:** Each cadence tick before `rebalancing` evaluates; also on adapter reconnect.
- **Goal:** Pull authoritative position + open-order snapshot from IBKR, compare to local `AccountPosition` aggregates, reconcile any drift (orphan fills, missed cancels, IBKR-side adjustments), emit `PositionUpdated` for any aggregate whose state changes.
- **Primary Aggregate:** `AccountPosition` (per ticker that drifts).
- **Pre-conditions:** IBKR session connected; latest internal state loadable.
- **Outcome (success):** Local state matches IBKR; if drift was detected and corrected, `PositionUpdated` emitted with reconciliation `reason` in audit log.
- **Outcome (failure):** IBKR snapshot fails → log; abort the trading-loop pass; `monitoring` raises a `critical` alert if persistent (per architecture.md `IbkrConnectionLost` policy).
- **Invariants Touched:** IBKR is the source of truth for `AccountPosition.quantity`; on conflict, IBKR wins and the local state is corrected with audit. Reconciliation must be idempotent — running it twice with no actual drift produces no events.
- **Cross-context Calls:** Calls IBKR adapter; emits `PositionUpdated` only on actual change.

### UC-6: EnforcePaperTradingFlag

- **Actor:** `ExecutionService` startup + every `SubmitRebalancePlan` call.
- **Trigger:** Process start; per-call validation in UC-1.
- **Goal:** Read `Configuration.paper_trading` (immutable per ADR-0002); if `true`, route all IBKR calls to the paper account; if `false`, require an explicit human-approved config flip (recorded via ADR per architecture.md §Key Constraints #4).
- **Primary Aggregate:** N/A — cross-cutting flag stamped on every `Order`.
- **Pre-conditions:** `Configuration` loaded.
- **Outcome (success):** Every `Order` carries `paper_trading: bool` for audit; IBKR connection uses the matching account credentials.
- **Outcome (failure):** Mismatch detected (e.g., `paper_trading=false` but credentials only allow paper) → fail-fast at startup, do not proceed to UC-1.
- **Invariants Touched:** `Order.paper_trading` is immutable once persisted; the flag matches the IBKR account that submitted the order; flipping the flag mid-run is forbidden (immutable `Configuration` per ADR-0002).
- **Cross-context Calls:** None — config-only and adapter-internal.

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| `Order.qty > 0` and `Order.side ∈ {BUY, SELL}` | `Order` | New |
| If `Order.order_type == LIMIT` then `limit_price > 0` and within fat-finger band of `LatestPriceView.last` | `Order` | New |
| `Σ fills.qty ≤ Order.qty` always | `Order` | New |
| Terminal statuses (`FILLED`, `REJECTED`, `CANCELLED`) admit no further state changes | `Order` | New |
| `(plan_id, action_index)` uniquely identifies an `Order` — duplicate `SubmitRebalancePlan` does NOT submit duplicate IBKR orders | `ExecutionService` | New (idempotency) |
| `AccountPosition.avg_cost` recomputed as weighted average on BUYs; unchanged on SELLs (v1 — no FIFO/lot tracking) | `AccountPosition` | New |
| IBKR is source of truth for `AccountPosition.quantity`; reconciliation overrides local on conflict | `AccountPosition` | New |
| Every `Order` records `paper_trading: bool` from `Configuration`; flag matches the IBKR account used | `Order` | New (anchor: ADR-0004 / architecture.md §Key Constraints #4) |
| Single IBKR account, USD-only in v1 (architecture.md §Key Constraints #6) | `ExecutionService` | New |
| This BC is the only one permitted to call IBKR (context-map dependency rule #1) | `ExecutionService` | Existing (architecture-level) |
| Cancellation is idempotent; multiple cancel requests collapse to one IBKR call | `Order` | New |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| Inbound | `rebalancing` | Sync command: `SubmitRebalancePlan(plan_id)` (ADR-0003) | Plan submission. |
| Inbound | `Configuration` (SK) | Shared kernel VOs | `paper_trading`, IBKR account selection. |
| Inbound | `market-data` | Read-model: `LatestPriceView` | Fat-finger sanity check on limit prices. |
| Inbound | (External: IBKR TWS / Client Portal) | ACL adapter `IbkrClient` | Order routing + fill / rejection callbacks. |
| Outbound | `monitoring` | Events: `OrderSubmitted`, `OrderFilled`, `OrderRejected`, `PositionUpdated` | Health metrics + alerts. |
| Outbound | `rebalancing` | Event: `OrderRejected` | Plan retry signal. |
| Outbound | `portfolio-construction` (next pass) | Read-model: `AccountPositionView` | Action classification + cap checks. |
| Outbound | `catalyst-timing` | Read-model: `AccountPositionView` | Held-name catalyst coverage. |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| IBKR ACL adapter (`IbkrClient`) | `execution` | TBD |
| `OrderStatus` translation table (IBKR ↔ internal) | `execution` | TBD |
| Pre-trade validators (qty, fat-finger, paper-trading flag) | `execution` | TBD |
| Reconciliation policy + audit | `execution` | TBD |
| `AccountPosition` cost-basis policy (v1: weighted avg) | `execution` | TBD |

## Constraints & Prohibited Actions

1. **Sole IBKR caller** — no other context may import or invoke `IbkrClient` (context-map dependency rule #1).
2. **No domain logic in the IBKR adapter** — adapter is pure ACL: vendor objects in, domain VOs out (architecture.md ACL row).
3. **`paper_trading` defaults to `true`** (ADR-0004 / architecture.md §Key Constraints #4); flipping to `false` requires a config change reviewed by a human via ADR.
4. **Synchronous response to `SubmitRebalancePlan`** (ADR-0003) — broker rejection / connection failure must propagate to the caller; do not buffer or fire-and-forget.
5. **Idempotency on `(plan_id, action_index)`** — duplicate sync commands MUST collapse to one IBKR submission per action.
6. **Single account, USD-only in v1** (architecture.md §Key Constraints #6); multi-account / multi-currency requires ADR.
7. **No silent fallbacks on IBKR errors** — every error code is either ACL-translated to a known `reason_code` (and surfaced via `OrderRejected`) or logged and surfaced as a `RiskAlertRaised` via `monitoring`.
8. **Audit every IBKR call** — append-only `audit/` jsonl with order id, prompt-style payload hash, and response (architecture.md §Cross-Cutting Concerns / Audit).
9. **No direct DB reads** of other contexts' tables (HR-1).

## Acceptance Criteria (testable)

1. Given a 3-action `RebalancePlan`, when `SubmitRebalancePlan(plan_id)` is invoked, then 3 `Order`s are persisted with status `SUBMITTED`, 3 `OrderSubmitted` events are emitted (one per order), and the sync return reports `{submitted: 3, rejected: 0}`.
2. Given the same `SubmitRebalancePlan(plan_id)` is called twice (e.g., orchestrator retry), when the second call runs, then no additional IBKR submissions occur and the sync return matches the first call's result (idempotency on `(plan_id, action_index)`).
3. Given an action with `limit_price` 20% above `LatestPriceView.last` (outside fat-finger band), when validating, then the action is rejected with `reason_code=FAT_FINGER`, `OrderRejected` is emitted, and IBKR is not called for that action.
4. Given a partial fill of 30 shares on an `Order(qty=100)`, when `HandleFillFromIbkr` runs, then `OrderStatus=PARTIALLY_FILLED`, one `Fill` is appended, `OrderFilled(is_complete=false, qty=30)` is emitted, and `AccountPosition.quantity` increases by 30 with `avg_cost` recomputed.
5. Given a duplicate IBKR fill notification (same `exec_id`), when handled, then no additional `Fill` is appended and no event is emitted (de-duplicated).
6. Given an IBKR rejection on a submitted `Order`, when `HandleRejectionFromIbkr` runs, then `OrderStatus=REJECTED`, `OrderRejected` is emitted with the IBKR-translated `reason_code`, and any subsequent IBKR callback for that id is logged as anomaly only.
7. Given `Configuration.paper_trading=true`, when `EnforcePaperTradingFlag` validates a submission, then the order is routed to the paper account and `Order.paper_trading=true` is recorded; given `paper_trading=false` without paper-allowing credentials, the process fails fast at startup.
8. Given local `AccountPosition.quantity=100` and IBKR snapshot reports 95 (orphan SELL outside the system), when `ReconcileWithIbkrAccount` runs, then local is corrected to 95, `PositionUpdated` is emitted, and the discrepancy is recorded in audit.
9. Given `Order.status=FILLED`, when a late fill callback arrives for that order, then it is logged as anomaly (no state change, no event), preserving the terminal-status invariant.

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | Fat-finger band — 5% of last price proposed; confirm? Should it widen during high volatility regimes? | Domain expert | Open |
| 2 | Cost-basis policy — weighted average for v1; lot-tracking (FIFO) deferred. Acceptable for paper-trading? | Domain expert | Open |
| 3 | `time_in_force` default — DAY (cancels at session close)? GTC? Per-action override? | Domain expert | Open |
| 4 | Do we need an `OrderCancelled` domain event? Currently UC-4 emits none; `monitoring` may want it for fill-latency / cancel-rate metrics. | Architecture + domain expert | Open |
| 5 | Reconciliation cadence — every cadence tick, or only on connect? Performance vs. correctness trade-off. | Domain expert | Open (lean: every tick + on reconnect) |
| 6 | IBKR connectivity — TWS or Client Portal Gateway? They have different callback semantics for fills. | Tech lead | Open |
| 7 | Position lock semantics — should a held ticker have a "rebalancing in progress" lock to prevent two overlapping plans (e.g., regime trigger + drift trigger) from compounding? | Domain expert | Open |
| 8 | Retry-on-`IbkrConnectionLost` — bounded exponential back-off, default max 60s per architecture.md. After 60s the trading-loop pass aborts; confirm? | Domain expert | Open |

## Approval

| Reviewer | Role | Decision | Date |
|----------|------|----------|------|
| TBD | Domain expert | Pending | — |

## Related Docs

| Doc | Path |
|-----|------|
| Source PRD(s) | [../../../../input/prd.md](../../../../input/prd.md) |
| Context Map | [../../../architecture/context-map.md](../../../architecture/context-map.md) |
| Architecture | [../../../architecture/architecture.md](../../../architecture/architecture.md) |
| Ubiquitous Language | [../../../architecture/ubiquitous-language.md](../../../architecture/ubiquitous-language.md) |
| Event Catalog | [../../../architecture/event-catalog.md](../../../architecture/event-catalog.md) |
| ADRs | [../../../architecture/adr/](../../../architecture/adr/) (esp. [0003-orchestrator-and-sync-rebalance.md](../../../architecture/adr/0003-orchestrator-and-sync-rebalance.md), [0004-direct-edgar-fetch-for-insider-and-13f.md](../../../architecture/adr/0004-direct-edgar-fetch-for-insider-and-13f.md)) |
