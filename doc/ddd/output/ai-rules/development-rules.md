# AI Development Rules

**Version:** 0.1
**Last Updated:** 2026-04-28
**Owner:** Architecture team

## Hard Rules (violation -> PR rejected)

| # | Rule | Rationale | Verification |
|---|------|-----------|--------------|
| HR-1 | No cross-domain direct database access. All cross-domain reads go through APIs, events, or materialized views. | Prevents hidden coupling between `signal-intelligence`, `portfolio-management`, `trade-operations`, `research-validation`. | Architecture lint (ruff custom rule + import-linter): imports of another domain's `domain.*` / `infra.*` modules fail CI. |
| HR-2 | No "shared common business model" packages. Shared kernels require an explicit ADR and are limited to value objects with no behaviour. | Today the only shared kernel is `Configuration` + primitives (`Ticker`, `Money`, `Weight`, `Percent`) under ADR-0002. Any new shared module needs its own ADR. | Build-time check: `src/core/` is allowlisted only if a matching `adr/NNNN-*-shared-kernel.md` exists with `Status: Accepted`; `src/core/` may not import from any `domains/`. |
| HR-3 | Contracts (`api-contract.yaml`, `events.asyncapi.yaml`) and implementations are updated in the same PR. | Prevents "spec drift" between the YAML wire format and the Python adapters. | CI fails if files in `src/<ctx>/` change without a contract diff in `domains/<ctx>/*.yaml` (or an ADR explaining why no contract change applies). |
| HR-4 | Aggregate invariants documented in `aggregate.md` must have at least one matching test. | Invariants without tests are aspirational; the model would silently rot. | Coverage check: each invariant ID (`INV-<ctx>-<n>`) maps to at least one test ID in `tests/invariants/`. |
| HR-5 | Ambiguous ownership requires an ADR before code is written. | Forces explicit decisions instead of "whoever sees it first" fixes (e.g., insider-cluster detection lives in `catalyst-timing`, not `market-data`). | PR template requires ADR link when the `ambiguous-ownership` label is set. |
| HR-6 | Every change must map to at least one acceptance criterion from a use case. | Ties code back to business intent (PRD §1.2). | PR description must reference `UC-<CTX>-<N>`; CI grep blocks merge otherwise. |
| HR-7 | Domain events are append-only — schema-breaking changes require a new event name and a deprecation window. | Protects subscribers (e.g., `monitoring` consumes from many publishers). | AsyncAPI diff lint (`spectral`) blocks breaking changes on existing channels; new event names are allowed any time. |
| HR-8 | LLM-driven agents must return JSON conforming to a published schema. Free-text replies are a hard error. | Per ADR-0005; the pipeline depends on deterministic, parseable outputs. | Each agent has a `pydantic` output model; runtime parse failure → `AgentParseError` logged with prompt-hash + response-hash; CI runs schema validation against recorded sample outputs. |
| HR-9 | All live IBKR order submission is gated by the `paper_trading` config flag. Flipping to live (`false`) requires a human-approved config change AND an ADR. | Per ADR-0004. Prevents accidental live trading. | CI check: any PR that defaults `paper_trading=false` in code, config, or test fixture is auto-rejected unless an ADR with `Status: Accepted` is linked. |
| HR-10 | `research-validation/backtesting` must never publish events on the production bus or call live broker adapters. | Backtests must not pollute live state or trigger real orders. | CI check: imports of `IbkrClient` or the production `EventBus` from `src/research_validation/` fail. Backtests use the `BacktestEventBus` and `MockIbkrClient` only. |

## AI Scope Rules (what the AI agent may and may not change)

| Allowed | Not Allowed (without explicit human approval) |
|---------|----------------------------------------------|
| Implement use cases inside one bounded context (`src/<domain>/<ctx>/`) | Modify another context's aggregate, domain service, or repository |
| Add/modify code under `src/<domain>/<ctx>/` if `domains/<ctx>/api-contract.yaml` and/or `events.asyncapi.yaml` are updated in the same PR | Edit a published `api-contract.yaml` or `events.asyncapi.yaml` to remove or rename a field on an existing operation/channel (HR-7) |
| Add tests for invariants (`tests/invariants/`) and acceptance criteria (`tests/unit/`, `tests/contract/`) | Delete or weaken an invariant test |
| Update the markdown contract `architecture/contracts/<a>-<b>.md` to reflect a YAML contract change | Introduce a new cross-context dependency without an integration-contract entry in `architecture/contracts/` |
| Refactor within an aggregate, including renaming internal helpers and splitting files | Move code across bounded-context boundaries |
| Update `domains/<ctx>/use-cases.md` Open Questions, add new ones, or change `Status: Draft (AI)` → `Draft (AI, revised)` | Mark a use case `Status: Approved` (only domain experts may; commit must be co-authored by the domain expert) |
| Add a new investment Framework strategy under `src/signal_intelligence/alpha_generation/frameworks/` (registered via the `Framework` interface) | Modify the `Framework` interface itself (changes contract for all frameworks → ADR required) |
| Implement an LLM agent's prompt template under `src/<ctx>/prompts/` and its `pydantic` output model | Increase LLM model tier (Sonnet → Opus) outside what's documented in `architecture.md` (cost impact → ADR) |
| Update the orchestrator's sequencing inside one pass | Add domain logic to the orchestrator (per the architecture rule: orchestrator may not implement domain logic) |
| Add structured log fields | Remove existing log fields (downstream dashboards depend on them) |

## Minimum Input Package

An AI agent must not start a code change unless **all** of these are present in the prompt or referenced files:

1. **Bounded context** name and link to `domains/<ctx>/bounded-context.md`.
2. **Business goal** — one sentence drawn from the source use case.
3. **Use case ID** (`UC-<CTX>-<N>`) and link to `domains/<ctx>/use-cases.md` (`Status: Approved`).
4. **Domain invariants** the change must preserve (list of `INV-<CTX>-<n>` IDs from `aggregates/*.md`).
5. **API / event contract** reference (`domains/<ctx>/api-contract.yaml#operationId` or event name in `architecture/event-catalog.md`).
6. **Acceptance criteria** — testable Given/When/Then.
7. **Impacted modules** — paths under `src/`.
8. **Non-functional requirements** — latency budget within the cadence interval; cost budget for LLM calls per pass; safety constraints (`paper_trading`, single-account).
9. **Prohibited actions** — from this rules file plus any use-case-specific constraints.
10. **Data sources** — which read-models / events the change consumes (must be documented in `context-map.md`).

If any item is missing, the AI must stop and request it instead of guessing (per WT-4 stop conditions).

## Repository Boundaries

| Path | Owner | AI may modify? |
|------|-------|----------------|
| `doc/ddd/output/architecture/` | Architecture team | Only via ADR proposal (`Status: Proposed`); never edit `context-map.md` / `architecture.md` / `ubiquitous-language.md` / `event-catalog.md` directly |
| `doc/ddd/output/architecture/adr/` | Architecture team | May propose new ADRs (status `Proposed`); accepted ADRs are immutable |
| `doc/ddd/output/architecture/contracts/` | Architecture team | May update markdown contracts when YAML contracts change in the same PR (HR-3) |
| `doc/ddd/output/domains/<ctx>/` | Context's owning team | Yes for `bounded-context.md`, `aggregates/*.md`, `api-contract.yaml`, `events.asyncapi.yaml`, within HR-1..HR-10 |
| `doc/ddd/output/domains/<ctx>/use-cases.md` | Domain expert | AI may add Open Questions and revise Drafts; only the domain expert may set `Status: Approved` |
| `doc/ddd/output/ai-rules/` | Architecture team | No — propose changes via ADR |
| `src/core/` (shared kernel) | Architecture team | No without ADR (HR-2); value-object additions only |
| `src/<domain>/<ctx>/` | Context's owning team | Yes, within HR-1..HR-10 |
| `src/orchestrator/` | Architecture team | Sequencing/scheduling changes yes; domain logic no |
| `src/adapters/` | Per-adapter `CODEOWNERS` (e.g., `IbkrClient` → trade-operations team) | Yes within the adapter; cross-adapter coupling no |
| `tests/` | Owning team | Yes |
| `data/` | Operations | No (read-only at runtime) |
| `CLAUDE.md` | Human developer (per CLAUDE.md itself) | **Never** — explicit project rule |

## Escalation

- **Conflict between two contexts** → open ADR (`Status: Proposed`); do not write code until accepted.
- **Governance rule appears to block a legitimate need** → ADR with `Governance Impact: Deviates` and a compensating control (e.g., temporary feature flag + monitoring alert).
- **AI cannot identify an owner** → stop; do not write code; surface to the architect.
- **LLM agent repeatedly returns invalid JSON (>2 retries)** → record `AgentParseError`, skip the affected pipeline step, raise a `warning` alert via `monitoring`. Do **not** fall back to free-text parsing.
- **IBKR rejects > N orders in one pass** → halt the loop, raise `critical` alert; require human acknowledgement before resuming.
- **`Configuration` change required mid-run** → forbidden. Stop the loop, change config, restart.

## Related Docs

| Doc | Path |
|-----|------|
| Review Checklist | [review-checklist.md](review-checklist.md) |
| Workflow Templates | [workflow-templates.md](workflow-templates.md) |
| Context Map | [../architecture/context-map.md](../architecture/context-map.md) |
| Architecture | [../architecture/architecture.md](../architecture/architecture.md) |
| Event Catalog | [../architecture/event-catalog.md](../architecture/event-catalog.md) |
| ADR Index | [../architecture/adr/](../architecture/adr/) |
