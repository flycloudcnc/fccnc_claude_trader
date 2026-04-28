# ADR-0003: Central `Trading Loop` Orchestrator and Synchronous `rebalancing → execution` Command

**Status:** Proposed
**Date:** 2026-04-28
**Deciders:** Architecture team
**Consulted:** Owners of `rebalancing`, `execution`, `monitoring`
**Informed:** All implementers

## Context

Two coordination questions arise in v1 and the PRD does not resolve them explicitly:

1. **How is the pipeline driven?** PRD §5 shows a `while market_open:` loop calling stages in order. Pure event-driven coordination (every stage reacts to upstream events) loses the deterministic ordering an operator needs to reason about a single daily/intra-day pass and makes "the loop ran successfully" undefinable.
2. **How does `rebalancing` hand work to `execution`?** The rest of the pipeline is async events, but a rebalance plan that fails broker validation must surface the failure *immediately* to the caller — not as an `OrderRejected` event minutes later — because the operator needs to know the plan was not actioned before the next loop cycle starts.

Both questions touch ownership and SLA semantics, which per the PDF require an ADR.

## Decision

We will introduce a **`Trading Loop` application service** (not a bounded context) at `src/orchestrator/` that drives the pipeline on a configurable cadence. It calls each stage in PRD §5 order, propagates a `run_id` for correlation, and isolates per-stage exceptions so a failure in one stage does not crash subsequent independent stages. **Asynchronous events remain the integration mechanism between most contexts**, but `rebalancing → execution` is a **synchronous command** (`SubmitRebalancePlan`) returning an `ExecutionAck` (`accepted`, `rejected_pre_trade`, `partial_failure`). All other context interactions stay event-driven.

## Scope

| Aspect | Value |
|--------|-------|
| Bounded Context(s) affected | All (orchestration); specifically `rebalancing` and `execution` for sync-command pattern |
| Aggregates affected | `RebalancePlan`, `Order`, `AccountPosition` |
| Contracts affected | `architecture/contracts/rebalancing-execution.md`; `domains/execution/api-contract.yaml` (sync) |
| Code modules affected | `src/orchestrator/*`, `src/portfolio_management/rebalancing/*`, `src/trade_operations/execution/*` |

## Options Considered

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | Pure event-driven (no orchestrator); every BC reacts to upstream events | Maximum decoupling; mirrors microservice ideal | No deterministic per-pass ordering; "did the daily loop run?" becomes a tracing exercise; backtesting reproducibility hurts |
| 2 | Orchestrator drives all stages; **all** stages including `rebalancing → execution` are async events | Uniform pattern; loose coupling | Operator can't tell within the same loop pass whether the plan was rejected; partial failures bleed across passes |
| 3 | **Orchestrator drives the pass; events between most stages; sync command for `rebalancing → execution` (Chosen)** | Deterministic loop; async decoupling where it helps; fail-fast where it matters | Mixed pattern requires explicit documentation; sync coupling between two contexts |
| 4 | No orchestrator; run each stage as an independent cron | Simplest ops | Loses run-id correlation; doubles operational complexity for backtesting |

## Consequences

**Positive**
- One `run_id` correlates every event/log line in a single trading-loop pass — reproducibility for postmortems and backtests.
- Operators see a single "loop pass succeeded / failed" signal.
- `rebalancing` can fail-fast on broker rejection within the same pass and emit a `RiskAlertRaised` for the operator.
- Sync coupling is **only** between `rebalancing` and `execution`; the rest of the system stays asynchronous.

**Negative**
- The orchestrator is a single point of failure for the loop. Mitigation: per-stage exception isolation and a separate health-heartbeat.
- Mixed sync/async pattern requires the contract docs to be explicit per relationship.
- `rebalancing` blocks while `execution` validates with IBKR — adds loop-pass latency. Mitigation: pre-trade validation is local-only (constraint check); IBKR submission itself is fire-and-forget (`OrderSubmitted` event).

**Neutral / Follow-up**
- Define the orchestrator's "stage failed" policy: which failures abort the pass, which continue. Initial rule: market-data failure aborts; downstream stage failure logs + skips remaining dependents only.
- Add a heartbeat event `LoopPassCompleted{run_id, duration_s, stage_outcomes}`.
- Decide later (separate ADR) whether to support running the loop on a cadence shorter than 1 minute.

## Governance Impact

| Governance Rule | Effect of this ADR |
|-----------------|--------------------|
| HR-1 (no cross-context DB reads) | Conforms — orchestrator only reads via published ports. |
| Cross-context contract introduction | Refines — establishes when sync vs. async is appropriate. |
| HR-3 (contracts linted in CI) | Conforms — sync command appears in `domains/execution/api-contract.yaml`; AsyncAPI for events. |

No deviation; this ADR establishes the rule that other contracts cite.

## Verification

- A contract test exercises `SubmitRebalancePlan` with an over-cap plan and asserts `rejected_pre_trade` is returned **synchronously** before any IBKR call.
- An integration test runs one full loop pass against a mock IBKR adapter and asserts a single `run_id` appears on every emitted event.
- The orchestrator's "stage failed" policy is unit-tested with deliberate mid-pipeline failures.
- PR review checklist: any new cross-context call must say "sync" or "async" and cite this ADR if sync.

## Related Docs

| Doc | Path |
|-----|------|
| Context Map (relationships table) | [../context-map.md](../context-map.md) |
| Architecture (Coordination row) | [../architecture.md](../architecture.md) |
| Event Catalog | [../event-catalog.md](../event-catalog.md) |
| Integration Contract (sync) | [../contracts/rebalancing-execution.md](../contracts/rebalancing-execution.md) |
| Development Rules | [../../ai-rules/development-rules.md](../../ai-rules/development-rules.md) |
| ADR-0001 (Modular monolith) | [0001-modular-monolith-hexagonal.md](0001-modular-monolith-hexagonal.md) |
