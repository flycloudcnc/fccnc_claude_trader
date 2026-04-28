# DDD Template: AI Workflow Templates (Prompt Library)

## Usage

Use this template to produce **`ai-rules/workflow-templates.md`** — the canonical prompt structures the team uses when invoking an AI agent for any of the six layers. The PDF requires that prompts ship the Minimum Input Package; these templates encode that.

One file per system. Lives at `ai-rules/workflow-templates.md`.

---

## Template

```markdown
# AI Workflow Templates

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

> Every AI invocation in this repo MUST start from one of these templates.
> Anything ad-hoc must be promoted to a template before reuse.

## How to use a template

1. Copy the template block.
2. Fill every `[bracketed]` slot. Empty slots are not allowed — if you can't fill one, the change isn't ready.
3. Reference files by repo-relative path; do not paste large blobs.
4. The agent must echo back the Minimum Input Package items it received before producing output.

---

## WT-1 — Intent Translation (Layer 1 -> 2)

```
Source PRD / Epic:
  [doc/prd.md#section-name]

Existing domain model (read these first):
  - architecture/context-map.md
  - architecture/bounded-contexts.md (if present)
  - domains/<ctx>/ubiquitous-language.md (per candidate context)

Task:
  Produce a Layer 2 intent-translation file using the
  `intent-translation.md` template. Output to:
    domains/<chosen-context>/use-cases.md

Required output sections:
  1. Affected bounded contexts (with primary/secondary)
  2. Term mapping (flag NEW terms)
  3. Use cases (actor, trigger, goal, primary aggregate, outcomes)
  4. Domain invariants (new vs existing)
  5. Cross-context dependencies
  6. Ownership
  7. Constraints / prohibited actions
  8. Acceptance criteria (testable)
  9. Open questions for the domain expert

Constraints:
  - Status MUST start as `Draft (AI)`.
  - Do not invent missing concepts; raise them as Open Questions.
  - Do not add a term to ubiquitous-language.md — only flag it as NEW.
```

---

## WT-2 — Domain Specification Generation (Layer 3)

```
Approved use cases:
  domains/<ctx>/use-cases.md  (Status: Approved)

Reference docs (load Layer 1 first):
  - architecture/context-map.md
  - architecture/architecture.md
  - domains/<ctx>/ubiquitous-language.md
  - architecture/event-catalog.md (summary only)

Task:
  Generate / update for context <ctx>:
    - domains/<ctx>/bounded-context.md
    - domains/<ctx>/aggregates/<aggregate>.md  (one per aggregate)
    - domains/<ctx>/invariants.md  (cross-aggregate rules only)

Constraints:
  - Follow `bounded-context.md` and `aggregate.md` templates literally.
  - Every invariant must be linked to a use case acceptance criterion.
  - Every command must produce at least one domain event.
  - Do not modify another context's docs.
```

---

## WT-3 — Contract Generation (Layer 4)

```
Inputs:
  - domains/<ctx>/bounded-context.md
  - domains/<ctx>/aggregates/*.md
  - domains/<ctx>/use-cases.md
  - architecture/event-catalog.md

Task:
  Produce / update:
    - domains/<ctx>/api-contract.yaml          (OpenAPI 3.1)
    - domains/<ctx>/events.asyncapi.yaml       (AsyncAPI 2.6)
    - domains/<ctx>/integration-contract.md    (markdown counterpart)
    - architecture/event-catalog.md            (add new events)

Constraints:
  - Each Command in bounded-context.md -> one OpenAPI operation.
  - Each Domain Event -> one AsyncAPI channel.
  - No breaking change on existing channels (HR-7).
  - Spectral lint must pass.
```

---

## WT-4 — Implementation (Layer 5)

```
MINIMUM INPUT PACKAGE (all required):
  1. Bounded context:        [domains/<ctx>/bounded-context.md]
  2. Business goal:          [one sentence]
  3. Use case:               [UC-N -> domains/<ctx>/use-cases.md#uc-N]
  4. Domain invariants:      [list IDs]
  5. API/event contract:     [domains/<ctx>/api-contract.yaml#operationId or event name]
  6. Acceptance criteria:    [list — Given/When/Then]
  7. Impacted modules:       [paths under services/ or apps/ or packages/]
  8. NFRs:                   [latency / throughput / security]
  9. Prohibited actions:     [from ai-rules/development-rules.md plus use-case-specific]

Task:
  Implement the use case end-to-end:
    - Code under the impacted modules
    - Tests: invariant + acceptance criteria + contract test
    - No changes outside the named modules without an ADR

Stop conditions (do NOT proceed; ask):
  - Any of items 1–9 is missing.
  - Acceptance criteria contradict an invariant.
  - The change requires touching another bounded context.
```

---

## WT-5 — Review (Layer 6)

```
PR diff:           [link]
Use case:          [UC-N]
Contracts changed: [list of YAML files]
Invariants tested: [list]

Task:
  Run `ai-rules/review-checklist.md` against the diff.
  For each unchecked box, cite the rule and the offending file/line.
  Return a single REJECT or APPROVE with rationale.

Constraints:
  - Do not auto-merge.
  - If any HR rule is violated, REJECT regardless of other quality.
```

---

## WT-6 — ADR Drafting

```
Trigger:
  [ambiguous-ownership | rule-deviation | new-shared-kernel | breaking-contract]

Inputs:
  - The use case or PR that triggered the question
  - Context Map and any affected bounded-context docs
  - ai-rules/development-rules.md

Task:
  Draft an ADR using `adr.md` template at:
    architecture/adr/NNNN-<slug>.md
  with Status: Proposed.

Constraints:
  - One decision per ADR.
  - At least three options in the Options Considered table.
  - If `Governance Impact: Deviates`, a compensating control is mandatory.
```

---

## Invocation Audit

Every AI invocation produced by these templates must be recorded in the activity log with:

| Field | Example |
|-------|---------|
| Template ID | WT-4 |
| Agent / model | claude-opus-4-7 |
| Inputs (hashes or paths) | UC-12, api-contract.yaml@a3f1 |
| Output PR / file | PR #234 |
| Reviewer | @alice |
| Date | 2026-04-28 |

This is the audit trail referenced in `development-rules.md` HR-1..HR-7.
```

---

## Guidelines

1. **Templates are mandatory** — anything ad-hoc must be promoted to a template before reuse.
2. **The Minimum Input Package is encoded literally** — WT-4 enforces it; do not relax.
3. **Stop conditions are first-class** — agents must surface missing inputs rather than guess (PDF: "AI is the drafter, not the decider").
4. **Audit log is non-optional** — without it, the governance loop is unverifiable.
5. **Versioned** — bumping a template requires an ADR if it changes the Minimum Input Package.
