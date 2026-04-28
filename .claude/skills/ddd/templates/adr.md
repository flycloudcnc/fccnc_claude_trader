# DDD Template: Architecture Decision Record (ADR)

## Usage

Use this template to record a single architecturally significant decision — especially when **ownership is ambiguous, a governance rule is being deviated from, or a cross-context impact requires explicit sign-off**. The PDF mandates an ADR for any ambiguous-ownership case; treat that as a hard rule.

One ADR per file. File name: `NNNN-<short-slug>.md` (zero-padded sequence). Lives at `architecture/adr/`.

ADRs are **append-only** by convention — never edit a `Accepted` ADR; supersede it with a new one and link both ways.

---

## Template

```markdown
# ADR-[NNNN]: [Short Title]

**Status:** [Proposed / Accepted / Superseded by ADR-XXXX / Deprecated]
**Date:** [YYYY-MM-DD]
**Deciders:** [Names / roles who agreed]
**Consulted:** [Domain experts / teams asked]
**Informed:** [Teams told after the fact]

## Context

[2-5 sentences. The forces at play: business pressure, conflicting requirements, technical constraint, ambiguous ownership, regulatory rule. Be specific — link to the PRD section or use case that triggered this decision.]

## Decision

[1-3 sentences in active voice. "We will ..." Make it implementable — a reader should know exactly what changes.]

## Scope

| Aspect | Value |
|--------|-------|
| Bounded Context(s) affected | [List] |
| Aggregates affected | [List] |
| Contracts affected | [List] |
| Code modules affected | [Paths] |

## Options Considered

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | [Option] | [Pros] | [Cons] |
| 2 | [Option] | [Pros] | [Cons] |
| 3 | [Chosen] | [Pros] | [Cons] |

## Consequences

**Positive**
- [What gets easier or safer]

**Negative**
- [What gets harder, what tech debt is taken on]

**Neutral / Follow-up**
- [Work this decision implies but does not yet do]

## Governance Impact

> Required when this ADR deviates from, refines, or interprets a governance rule.

| Governance Rule | Effect of this ADR |
|-----------------|--------------------|
| [Rule text or ID from `ai-rules/development-rules.md`] | [Conforms / Deviates / Refines] |

If **Deviates**: include the compensating control (extra test, manual review, time-boxed exception).

## Verification

- [How we will know this decision is being followed: contract test, architecture lint, PR checklist item, etc.]

## Related Docs

| Doc | Path |
|-----|------|
| Triggering Use Case | [domains/<context>/use-cases.md#uc-N] |
| Context Map | [../context-map.md] |
| Bounded Context | [../../domains/<context>/bounded-context.md] |
| Development Rules | [../../ai-rules/development-rules.md] |
| Superseded ADR(s) | [NNNN-...md] |
```

---

## Guidelines

1. **One decision per ADR** — if you find yourself writing "and also...", split it.
2. **Status transitions are explicit** — `Proposed -> Accepted` requires the listed deciders' agreement; `Accepted -> Superseded` requires a new ADR that points back.
3. **Append-only** — accepted ADRs are historical record. Fix mistakes by superseding, not editing.
4. **Mandatory triggers** — write an ADR when (a) ownership is ambiguous, (b) a governance rule is deviated from, (c) a cross-context contract is introduced or breaks compatibility, (d) a shared kernel is proposed.
5. **Governance Impact section is required** for any rule deviation — without a compensating control, the deviation should be rejected.
6. **Keep under ~120 lines** — long ADRs hide the decision. Push detail into the linked specs.
7. **Link from the use case** — every use case whose ambiguity led to an ADR must link back.
