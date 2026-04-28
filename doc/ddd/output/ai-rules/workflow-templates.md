# AI Workflow Templates

**Version:** 0.1
**Last Updated:** 2026-04-28

> Every AI invocation in this repo MUST start from one of these templates.
> Anything ad-hoc must be promoted to a template before reuse.

## How to use a template

1. Copy the template block.
2. Fill every `[bracketed]` slot. Empty slots are not allowed — if you can't fill one, the change isn't ready.
3. Reference files by repo-relative path; do not paste large blobs.
4. The agent must echo back the Minimum Input Package items it received before producing output.

---

## WT-1 — Intent Translation (Layer 1 → 2)

```
Source PRD / Epic:
  [doc/ddd/input/prd.md#section-name]
  [doc/ddd/input/investment-frameworks.md#framework-name (if applicable)]

Existing domain model (read these first):
  - doc/ddd/output/architecture/context-map.md
  - doc/ddd/output/architecture/ubiquitous-language.md
  - doc/ddd/output/domains/<ctx>/ubiquitous-language.md (if context-local overrides exist)

Task:
  Produce a Layer 2 intent-translation file using the
  `intent-translation.md` template. Output to:
    doc/ddd/output/domains/<chosen-domain>/<chosen-context>/use-cases.md

Required output sections:
  1. Affected bounded contexts (with primary/secondary)
  2. Term mapping (flag NEW terms not yet in ubiquitous-language.md)
  3. Use cases (actor, trigger, goal, primary aggregate, outcomes) — labelled UC-<CTX>-<N>
  4. Domain invariants (new vs existing) — labelled INV-<CTX>-<n>
  5. Cross-context dependencies
  6. Ownership
  7. Constraints / prohibited actions
  8. Acceptance criteria (testable Given/When/Then)
  9. Open questions for the domain expert

Constraints:
  - Status MUST start as `Draft (AI)`.
  - Do not invent missing concepts; raise them as Open Questions.
  - Do not add a term to architecture/ubiquitous-language.md — only flag it as NEW.
  - Do not generate any Layer 3 docs (bounded-context.md, aggregates) in the same session.
```

---

## WT-2 — Domain Specification Generation (Layer 3)

```
Approved use cases:
  doc/ddd/output/domains/<domain>/<ctx>/use-cases.md  (Status: Approved)

Reference docs (load Layer 1 first):
  - doc/ddd/output/architecture/context-map.md
  - doc/ddd/output/architecture/architecture.md
  - doc/ddd/output/architecture/ubiquitous-language.md
  - doc/ddd/output/architecture/event-catalog.md (summary only)

Task:
  Generate / update for context <domain>/<ctx>:
    - doc/ddd/output/domains/<domain>/<ctx>/bounded-context.md
    - doc/ddd/output/domains/<domain>/<ctx>/aggregates/<aggregate>.md  (one per aggregate)
    - doc/ddd/output/domains/<domain>/<ctx>/invariants.md  (cross-aggregate rules only)
    - doc/ddd/output/domains/<domain>/<ctx>/ubiquitous-language.md  (overrides only, optional)

Constraints:
  - Follow `bounded-context.md` and `aggregate.md` templates literally.
  - Every invariant must be linked to a use case acceptance criterion (HR-4 + HR-6).
  - Every command must produce at least one domain event.
  - Every domain event added must also be added to architecture/event-catalog.md in the same PR.
  - Do not modify another context's docs.
  - One aggregate per session is preferred (context-window discipline).
```

---

## WT-3 — Contract Generation (Layer 4)

```
Inputs:
  - doc/ddd/output/domains/<domain>/<ctx>/bounded-context.md
  - doc/ddd/output/domains/<domain>/<ctx>/aggregates/*.md
  - doc/ddd/output/domains/<domain>/<ctx>/use-cases.md
  - doc/ddd/output/architecture/event-catalog.md

Task:
  Produce / update:
    - doc/ddd/output/domains/<domain>/<ctx>/api-contract.yaml          (OpenAPI 3.1)
    - doc/ddd/output/domains/<domain>/<ctx>/events.asyncapi.yaml       (AsyncAPI 2.6)
    - doc/ddd/output/architecture/contracts/<upstream>-<downstream>.md (markdown counterpart)
    - doc/ddd/output/architecture/event-catalog.md                     (add new events)

Constraints:
  - Each Command in bounded-context.md → one OpenAPI operation (operationId = command name).
  - Each Domain Event → one AsyncAPI channel.
  - No breaking change on existing channels or operations (HR-7).
  - Spectral lint must pass (`spectral lint <file>`).
  - One contract pair per session (markdown + YAML).
```

---

## WT-4 — Implementation (Layer 5)

```
MINIMUM INPUT PACKAGE (all required — stop if any is missing):
  1.  Bounded context:        [doc/ddd/output/domains/<domain>/<ctx>/bounded-context.md]
  2.  Business goal:          [one sentence from the source use case]
  3.  Use case:               [UC-<CTX>-<N> → doc/ddd/output/domains/<domain>/<ctx>/use-cases.md#uc-CTX-N]
  4.  Domain invariants:      [INV-<CTX>-<n>, ... from aggregates/<agg>.md]
  5.  API/event contract:     [domains/<domain>/<ctx>/api-contract.yaml#operationId | event name]
  6.  Acceptance criteria:    [list — Given/When/Then]
  7.  Impacted modules:       [paths under src/<domain>/<ctx>/ or src/orchestrator/ or src/adapters/]
  8.  NFRs:                   [loop-budget impact (must keep loop < 80% of cadence);
                               LLM cost-per-pass impact;
                               safety: paper_trading must remain true (HR-9);
                               single-account, USD-only constraints]
  9.  Prohibited actions:     [from doc/ddd/output/ai-rules/development-rules.md plus use-case-specific]
  10. Data sources:           [read-models / events consumed (must be in context-map.md)]

Task:
  Implement the use case end-to-end:
    - Code under the impacted modules
    - Tests: invariant + acceptance criteria + contract test (in tests/ subtree, run in .venv)
    - Update domains/<ctx>/api-contract.yaml or events.asyncapi.yaml in the same PR if behaviour visible to other contexts changed (HR-3)
    - Update activity-log entry at the top of the activity-log file (per CLAUDE.md)
    - No changes outside the named modules without an ADR

Stop conditions (do NOT proceed; ask):
  - Any of items 1–10 is missing.
  - Acceptance criteria contradict an invariant.
  - The change requires touching another bounded context.
  - The change defaults `paper_trading=false` anywhere.
  - The change introduces multi-account or non-USD code paths.
  - LLM agent does not have a pydantic output model defined (HR-8).

Standard agent / model:
  - Implementation: claude-opus-4-7 (or claude-sonnet-4-6 for routine refactors)
  - Use the Anthropic prompt cache for the `architecture/*.md` and `ai-rules/*.md` prefix.
```

---

## WT-5 — Review (Layer 6)

```
PR diff:           [link]
Use case:          [UC-<CTX>-<N>]
Contracts changed: [list of YAML files]
Invariants tested: [list of INV-<CTX>-<n>]

Task:
  Run doc/ddd/output/ai-rules/review-checklist.md against the diff.
  For each unchecked box, cite the rule (HR-N) and the offending file/line.
  Return a single REJECT or APPROVE with rationale.

Constraints:
  - Do not auto-merge.
  - If any HR rule is violated, REJECT regardless of other quality.
  - If `paper_trading=false` appears anywhere new (HR-9), REJECT immediately.
  - If a backtest module imports IbkrClient or production EventBus (HR-10), REJECT immediately.
```

---

## WT-6 — ADR Drafting

```
Trigger:
  [ambiguous-ownership | rule-deviation | new-shared-kernel | breaking-contract | live-trading-flip | model-tier-upgrade]

Inputs:
  - The use case or PR that triggered the question
  - doc/ddd/output/architecture/context-map.md and any affected bounded-context docs
  - doc/ddd/output/ai-rules/development-rules.md

Task:
  Draft an ADR using the `adr.md` template at:
    doc/ddd/output/architecture/adr/NNNN-<slug>.md
  with Status: Proposed.

Constraints:
  - One decision per ADR.
  - At least three options in the Options Considered table.
  - If `Governance Impact: Deviates`, a compensating control is mandatory.
  - Must cite the HR rule(s) being touched.
  - Do not mark Status: Accepted in the same PR — separate human approval required.
```

---

## WT-7 — Backtest Execution (research-validation/backtesting)

```
Inputs:
  1. Strategy config:         [path to strategy hash / config]
  2. Backtest window:         [start_date → end_date]
  3. Universe:                [US equities only, decision #8]
  4. Frameworks enabled:      [list]
  5. Constraints applied:     [PRD §4.5 set + any overrides]

Task:
  Run a backtest via src/research_validation/backtesting/.
  Produce:
    - BacktestRun aggregate persisted
    - BacktestCompleted event on the BACKTEST event bus (NOT production bus, HR-10)
    - Summary metrics: CAGR, max drawdown, Sharpe, hit rate, turnover

Stop conditions:
  - Window crosses into the future.
  - Any code path attempts to call IbkrClient (HR-10 violation).
  - Any code path attempts to publish on the production EventBus (HR-10 violation).

Standard agent / model:
  - Use the same agent versions documented in architecture.md for live runs.
  - Record agent version in BacktestRun for reproducibility.
```

---

## Invocation Audit

Every AI invocation produced by these templates must be recorded in the activity log with:

| Field | Example |
|-------|---------|
| Template ID | WT-4 |
| Agent / model | claude-opus-4-7 |
| Inputs (hashes or paths) | UC-EX-1, api-contract.yaml@a3f1, INV-EX-1, INV-EX-2 |
| LLM prompt hash (for HR-8 agents) | sha256:9f… |
| LLM response hash | sha256:c1… |
| Output PR / file | PR #234 |
| Reviewer | @alice |
| Date | 2026-04-28 |

This is the audit trail referenced in `development-rules.md` HR-1..HR-10.

## Related Docs

| Doc | Path |
|-----|------|
| Development Rules | [development-rules.md](development-rules.md) |
| Review Checklist | [review-checklist.md](review-checklist.md) |
| Context Map | [../architecture/context-map.md](../architecture/context-map.md) |
| Architecture | [../architecture/architecture.md](../architecture/architecture.md) |
| Event Catalog | [../architecture/event-catalog.md](../architecture/event-catalog.md) |
