# AI-Assisted PR Review Checklist

**Version:** 0.1
**Last Updated:** 2026-04-28

> Reviewers: tick every box. An unticked box blocks merge.
> AI agents: run this checklist *before* opening the PR; cite each item in the PR description.

## 1. Intent Traceability

- [ ] PR description names a Use Case ID (`UC-<CTX>-<N>`) from `domains/<ctx>/use-cases.md` (HR-6).
- [ ] The use case is `Status: Approved` (not `Draft (AI)`).
- [ ] Every acceptance criterion in the use case has a corresponding test in this PR.
- [ ] Open Questions on the use case are all `Answered`.
- [ ] PR description links to the source PRD section (e.g., `doc/ddd/input/prd.md#4.1`) where applicable.

## 2. Domain Boundaries

- [ ] All modified `src/` files live under a single `src/<domain>/<ctx>/` (or `src/core/` / `src/orchestrator/` / `src/adapters/` with ADR).
- [ ] No imports from another `src/<other-domain>/<other-ctx>/` module (HR-1).
- [ ] No new `src/core/*` (shared kernel) directory or symbol (HR-2) — or an ADR is linked.
- [ ] `src/research_validation/backtesting/` does not import `IbkrClient` or the production `EventBus` (HR-10).
- [ ] Cross-context interactions go through the published contract; no direct DB or in-process cross-context calls.
- [ ] `src/orchestrator/` contains no domain logic (only sequencing, scheduling, error escalation).

## 3. Contracts

- [ ] If behaviour visible to other contexts changed, `domains/<ctx>/api-contract.yaml` and/or `events.asyncapi.yaml` were updated in **this** PR (HR-3).
- [ ] OpenAPI 3.1 / AsyncAPI 2.6 lints clean (`spectral lint`).
- [ ] No breaking change on an existing event channel or operation (HR-7) — additions only, or new event/operation name with deprecation note.
- [ ] Markdown `architecture/contracts/<a>-<b>.md` reflects the YAML diff.
- [ ] Every new event payload field appears in `architecture/event-catalog.md` and the publishing context's `events.asyncapi.yaml`.
- [ ] Every new event has at least one declared subscriber (no orphan events).

## 4. Aggregates & Invariants

- [ ] All writes go through an aggregate root; no external code reaches into child entities.
- [ ] Each invariant in the aggregate doc has a test (HR-4); diff includes test ID → invariant ID mapping.
- [ ] State-machine transitions in `aggregate.md` match the code's allowed transitions.
- [ ] Concurrency strategy (optimistic locking via aggregate version) is preserved.
- [ ] No two aggregates updated in the same DB transaction (one-aggregate-per-transaction rule).

## 5. Ubiquitous Language

- [ ] All new identifiers use canonical names from `architecture/ubiquitous-language.md` (or the per-context override).
- [ ] New terms introduced in this PR were added to the glossary.
- [ ] No banned aliases (`Symbol`, `Stock`, `Idea`, `Pick`, `Allocation`, `Trade list`, `Strategy` for `Framework`, etc.) reintroduced.
- [ ] Ambiguous terms (`Position`, `Signal`, `Risk`, `Score`, `Insider`, "Rebalance" as verb) are qualified per the glossary's "Ambiguous Terms to Avoid" table.

## 6. Tests

- [ ] Unit tests for new domain logic (`tests/unit/<ctx>/`).
- [ ] Contract tests against the YAML for any new endpoint or event (`tests/contract/`).
- [ ] Invariant tests for any new or changed invariant (`tests/invariants/`).
- [ ] Coverage on changed lines ≥ project threshold.
- [ ] Tests run in `.venv` (per CLAUDE.md project rule).

## 7. LLM Agent Discipline (HR-8)

- [ ] Every new LLM call has a `pydantic` output model under `src/<ctx>/prompts/`.
- [ ] Prompt template is versioned (file path + git hash recorded in audit log).
- [ ] Free-text fallback is **not** present; parse failures raise `AgentParseError` and log prompt-hash + response-hash.
- [ ] Model tier (Sonnet vs. Opus) matches the choice documented in `architecture.md`; tier upgrades have an ADR.
- [ ] Anthropic prompt cache is used where the prompt prefix is stable across many tickers.

## 8. Safety (HR-9 — IBKR / live trading)

- [ ] No PR defaults `paper_trading=false` in code, config, fixture, or test (HR-9) without an Accepted ADR linked.
- [ ] No new direct call to `IbkrClient.place_order(...)` outside `src/trade_operations/execution/`.
- [ ] Multi-account or non-USD code paths are not introduced (decisions #9, #12) — or an ADR is linked.

## 9. Governance & ADRs

- [ ] If the change touches ownership, shared kernel, or any HR rule deviation, an ADR is linked (status `Proposed` or `Accepted`).
- [ ] If an ADR is `Proposed`, the PR is marked Draft until accepted.
- [ ] No edit to an `Accepted` ADR (immutable; supersede instead).
- [ ] No edit to `CLAUDE.md` (project rule: human developer only).

## 10. AI Audit Trail

- [ ] PR description names the agent / model used (e.g., `claude-opus-4-7`).
- [ ] PR description lists the Minimum Input Package items (links accepted) — confirms HR pre-conditions were met.
- [ ] Generated code was reviewed by a human, not auto-merged.
- [ ] No "TODO from AI" left in code without an issue link.
- [ ] LLM prompt + response hashes recorded in the audit log per WT-Invocation Audit.

## 11. Non-functional

- [ ] Loop-budget impact assessed: the change does not push a single Trading-Loop pass past 80% of the cadence interval (otherwise raises `LoopOverrun`).
- [ ] LLM cost-per-pass impact noted in PR description if the change adds an agent call.
- [ ] Logging follows the architecture doc's logging convention (`structlog` JSON, mandatory keys `run_id`, `context`, `ticker`).
- [ ] No new secret literals; all credentials via the configured secret store.

## 12. Rollout

- [ ] Migration plan documented if the change includes schema migrations.
- [ ] Feature flag or staged rollout if the change is risky and the use case allows.
- [ ] Rollback strategy is at least one sentence.
- [ ] Activity-log entry added at the top of the activity-log file (per CLAUDE.md).

## Sign-off

| Role | Name | Date |
|------|------|------|
| Code reviewer | | |
| Domain expert (for cross-context changes or `Status: Approved` flips) | | |
| Architect (for ADR-bearing changes) | | |

## Related Docs

| Doc | Path |
|-----|------|
| Development Rules | [development-rules.md](development-rules.md) |
| Workflow Templates | [workflow-templates.md](workflow-templates.md) |
| Context Map | [../architecture/context-map.md](../architecture/context-map.md) |
| Architecture | [../architecture/architecture.md](../architecture/architecture.md) |
