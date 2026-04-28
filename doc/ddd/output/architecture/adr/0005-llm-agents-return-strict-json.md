# ADR-0005: LLM Agents Return Strict JSON Parsed into Domain Value Objects

**Status:** Proposed
**Date:** 2026-04-28
**Deciders:** Architecture team
**Consulted:** Owners of `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`, `monitoring`
**Informed:** All implementers, prompt authors

## Context

PRD §11 mandates "Strict JSON outputs (no free text)" and "Deterministic agent interfaces" for all agents. The investment-frameworks document (`doc/ddd/input/investment-frameworks.md`) is written as 12 prose Claude prompts that return narrative reports. The two are in tension: the frameworks are authored as research prose, but the pipeline contracts demand machine-parseable structured output. Inside the pipeline, agent output is consumed by downstream BCs as events (`AlphaCandidateGenerated`, `ConvictionAssessed`, `CatalystIdentified`, `RiskRegimeChanged`) whose AsyncAPI payloads are already typed. If the LLM returns prose we must re-parse with a second pass, doubling cost and adding non-determinism.

## Decision

Every LLM-driven agent (any code that calls Claude as part of a domain operation) **must** call the Anthropic API with **`tool_use` or structured-output mode** producing **JSON that validates against a JSON Schema co-located with the agent**. The parsed JSON is then constructed into the appropriate domain Value Object inside the BC; raw LLM strings never cross a context boundary. Each agent owns one schema file (`schema.json`) and one prompt file (`prompt.md`), both versioned. Schema validation failures are domain errors and emit a `RiskAlertRaised` event.

## Scope

| Aspect | Value |
|--------|-------|
| Bounded Context(s) affected | `alpha-generation`, `conviction-sizing`, `catalyst-timing`, `risk-regime`, `monitoring` (any BC using LLM reasoning) |
| Aggregates affected | `AlphaCandidate`, `StrategyRun`, `ConvictionAssessment`, `CatalystAssessment`, `RegimeState`, `RiskAlert` |
| Contracts affected | All `events.asyncapi.yaml` files in the affected contexts |
| Code modules affected | Each affected BC's `agents/<framework>/{prompt.md, schema.json, agent.py}` |

## Options Considered

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | Free-form prose output, parsed downstream by regex / LLM-as-parser | Mirrors framework prompts as written | Non-deterministic; double LLM cost; impossible to write contract tests |
| 2 | **Strict JSON via tool-use mode + per-agent schema (Chosen)** | Deterministic; single LLM call; contract-testable; aligns with existing AsyncAPI types | Prompts must be rewritten from prose into structured form; schema maintenance overhead |
| 3 | Free-form first call, structured second-call extraction | Less prompt rewriting | 2× cost and latency; introduces a second failure mode |

## Consequences

**Positive**
- LLM output is validated at the BC boundary — invalid responses never enter the domain model.
- Schemas double as documentation and test fixtures.
- Caching: identical (prompt, schema, input) tuples can be deterministically cached.
- AsyncAPI event payloads can be derived mechanically from agent schemas where shapes match.

**Negative**
- Every framework prompt from `investment-frameworks.md` must be re-authored into a structured prompt + schema pair. The narrative quality of the original prompts becomes commentary inside the prompt, not the output format.
- A schema change is a breaking change for downstream BCs and may require an event version bump (`EventNameV2`).
- Tool-use / structured-output mode constrains model choice — verify both Opus and Sonnet support the chosen mechanism for the model versions pinned in `architecture.md`.

**Neutral / Follow-up**
- Add a `tools/validate_agent_outputs.py` CLI that replays N historical LLM responses against the current schema to catch regressions.
- Decide later whether to use Anthropic's `tool_use` API or a JSON-Schema constrained sampling library — both satisfy this ADR; pick per-agent.
- Schema versioning convention: every schema has a top-level `$id` ending in `vN`; bumping N requires an event-catalog update.

## Governance Impact

| Governance Rule | Effect of this ADR |
|-----------------|--------------------|
| HR-3 (contracts linted in CI) | Conforms — agent schemas are linted with the same `spectral` / `ajv` step as OpenAPI/AsyncAPI. |
| HR-1 / HR-2 | Conforms — LLM output is converted to domain VOs at the BC boundary; no LLM strings leak into the kernel or other BCs. |
| "Deterministic agent interfaces" (PRD §11) | Conforms — strict JSON + schema makes interfaces deterministic; this ADR is the implementation. |

No deviation.

## Verification

- A contract test per agent: load `prompt.md` + `schema.json`, call the LLM with a fixture input, assert response validates against the schema.
- A unit test asserts each agent's output schema is referenced by at least one event in `events.asyncapi.yaml` (or marked `internal-only`).
- CI gate: any PR that adds an LLM call must include a co-located `prompt.md` + `schema.json`.
- PR review checklist: "every LLM call is structured-output / tool-use, never free text."

## Related Docs

| Doc | Path |
|-----|------|
| Architecture (Agent style row) | [../architecture.md](../architecture.md) |
| Event Catalog | [../event-catalog.md](../event-catalog.md) |
| Investment Frameworks (input prose) | [../../input/investment-frameworks.md](../../input/investment-frameworks.md) |
| Development Rules | [../../ai-rules/development-rules.md](../../ai-rules/development-rules.md) |
| Workflow Templates | [../../ai-rules/workflow-templates.md](../../ai-rules/workflow-templates.md) |
| ADR-0001 (Modular monolith) | [0001-modular-monolith-hexagonal.md](0001-modular-monolith-hexagonal.md) |
