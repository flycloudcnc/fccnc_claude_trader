# ADR-0002: `Configuration` as Shared Kernel — Value Objects Only

**Status:** Proposed
**Date:** 2026-04-28
**Deciders:** Architecture team
**Consulted:** Owners of every BC that consumes Configuration
**Informed:** All implementers

## Context

The PRD §7 defines a small set of user-supplied configuration values that constrain almost every stage of the pipeline: `risk_tolerance`, `target_return`, `max_drawdown`, `horizon`, `portfolio_size`. These same values appear in alpha screening filters (§4.1), conviction sizing (§4.2), risk-regime caps (§4.4), portfolio constraints (§4.5), and rebalance triggers (§4.6) — i.e. they cross **6 of the 10 bounded contexts**. The PDF model treats any model that crosses BC boundaries as needing an explicit decision: either duplicate (with translation), or formalize a Shared Kernel. Duplicating five primitive value objects across 6 contexts produces drift bugs (e.g. `MaxDrawdown(0.15)` vs. `MaxDrawdown(15.0)` semantic mismatch) and adds no isolation benefit because the VOs have no behavior.

## Decision

We will define a **Shared Kernel** at `src/core/` containing **only value objects**: `RiskTolerance`, `TargetReturn`, `MaxDrawdown`, `Horizon`, `PortfolioSize`, plus the primitive VOs `Ticker`, `Money`, `Weight`, `Percent`. The kernel contains **no aggregates, no repositories, no services, no I/O, no behavior beyond pure validation in `__init__`**. Every consuming BC imports these VOs directly. Any change to the kernel requires a new ADR superseding this one.

## Scope

| Aspect | Value |
|--------|-------|
| Bounded Context(s) affected | `alpha-generation`, `conviction-sizing`, `risk-regime`, `portfolio-construction`, `rebalancing`, `monitoring` (Configuration consumers); all contexts (primitive VOs) |
| Aggregates affected | All — VOs appear inside aggregate fields |
| Contracts affected | All OpenAPI/AsyncAPI schemas reference shared types |
| Code modules affected | `src/core/config/*`, `src/core/primitives/*` |

## Options Considered

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | Duplicate VOs in each context (Conformist or per-context redefinition) | No coupling; each BC fully owns its model | 6× drift surface for primitive concepts; bugs from semantic mismatch; no compile-time guarantee they mean the same thing |
| 2 | A "common" service called by every BC | Centralized validation | Adds a network hop for what is effectively a struct literal; LLM JSON parsing already covers validation; ridiculous overhead |
| 3 | **Shared Kernel of VOs only (Chosen)** | One source of truth; no behavior to drift; HR-2 makes the rule enforceable | Coupling — every BC depends on `src/core/`; any breaking change to a VO touches every consumer |

## Consequences

**Positive**
- A single `MaxDrawdown(0.15)` means the same thing everywhere — represented as a `Decimal` in the range `(0, 1)`.
- OpenAPI/AsyncAPI schemas can `$ref` a shared `components/schemas/Money.yaml` etc., reducing wire-format drift.
- Tests for VO invariants live in one place.

**Negative**
- Coupling: a breaking change to any kernel VO requires coordinated updates across 6 BCs and a superseding ADR.
- Risk of "kernel creep" — contributors will want to put `RiskRegimeEvaluator` in `core/` because "everyone uses it." The HR-2 rule and PR review gate are the only defense.

**Neutral / Follow-up**
- Tag every kernel module with a `# SHARED KERNEL — see ADR-0002` header comment.
- Add an `import-linter` rule: nothing in `src/core/` may import from any BC package.
- Add an `import-linter` rule: every BC may import from `src/core/` but only from the `config/` and `primitives/` subpackages.

## Governance Impact

| Governance Rule | Effect of this ADR |
|-----------------|--------------------|
| HR-2 (Shared Kernel limited to immutable VOs; no entities, no services) | Conforms — kernel is VOs only and verified by import-linter. |
| HR-1 (no cross-context DB reads) | Conforms — kernel has no persistence. |
| HR-3 (contracts linted in CI) | Conforms — kernel schemas referenced in OpenAPI/AsyncAPI; lint catches drift. |

No deviation; this ADR **establishes** the kernel that HR-2 governs.

## Verification

- `import-linter` contract: `src.core.*` may not import from any `src.signal_intelligence`, `src.portfolio_management`, `src.trade_operations`, or `src.research_validation` package.
- `import-linter` contract: each BC may only import `src.core.config`, `src.core.primitives`, and `src.core.events` from the kernel.
- A unit-test suite (`tests/core/test_invariants.py`) asserts that every kernel VO is frozen (`@dataclass(frozen=True)`) and rejects out-of-range values via `__post_init__`.
- Any PR that adds a class outside `core/config/` or `core/primitives/` to `src/core/` triggers a CI check that requires a referenced superseding ADR.

## Related Docs

| Doc | Path |
|-----|------|
| Context Map (Shared Models section) | [../context-map.md](../context-map.md) |
| Architecture | [../architecture.md](../architecture.md) |
| Ubiquitous Language | [../ubiquitous-language.md](../ubiquitous-language.md) |
| Development Rules (HR-2) | [../../ai-rules/development-rules.md](../../ai-rules/development-rules.md) |
| ADR-0001 (Modular monolith) | [0001-modular-monolith-hexagonal.md](0001-modular-monolith-hexagonal.md) |
