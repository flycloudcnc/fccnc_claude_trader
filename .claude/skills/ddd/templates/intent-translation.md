# DDD Template: Intent Translation (Use Cases)

## Usage

Use this template to capture **Layer 2 — Intent Translation** for one bounded context: the bridge between business intent (epics, PRDs, stakeholder asks) and the domain specification. AI drafts this; a domain expert validates it before it feeds Layer 3.

One file per bounded context. Lives at `domains/<context>/use-cases.md`.

---

## Template

```markdown
# Intent Translation: [Bounded Context]

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]
**Source PRDs / Epics:** [paths or links]
**Domain Expert (validator):** [name / role]
**Status:** [Draft (AI) / Reviewed / Approved]

## Business Intent Summary

[2-4 sentences in business language — what stakeholders asked for, why, what outcome they expect.]

## Affected Bounded Contexts

| Context | Primary? | Reason |
|---------|----------|--------|
| [Context] | [Yes/No] | [Why this context is touched] |

> Primary = owns the change. Secondary = consumes / reacts.

## Term Mapping (Business -> Ubiquitous Language)

| Business Term | Ubiquitous Term | Code Name | New? |
|---------------|-----------------|-----------|------|
| [Stakeholder phrase] | [Canonical term in `ubiquitous-language.md`] | [PascalCase code] | [Yes if newly introduced] |

> Flag every term that does not yet exist in the glossary as **New**. Approved new terms must be added to `ubiquitous-language.md` before Layer 3 generation.

## Use Cases

### UC-[N]: [Use Case Name]

- **Actor:** [Role / system that initiates]
- **Trigger:** [Event or command that starts the flow]
- **Goal:** [What the actor wants, in domain language]
- **Primary Aggregate:** [Aggregate root that owns the state change]
- **Pre-conditions:** [What must already be true]
- **Outcome (success):** [Observable end state]
- **Outcome (failure):** [What happens on each failure mode]
- **Invariants Touched:** [Domain rules this use case must preserve]
- **Cross-context Calls:** [Events emitted, APIs called — by name only]

[Repeat for each use case in this PRD slice.]

## Domain Invariants Introduced or Affected

| Invariant | Scope (Aggregate / Context) | New / Existing |
|-----------|----------------------------|----------------|
| [Rule that must always hold] | [Where it applies] | [New / Existing] |

## Cross-Context Dependencies

| Direction | Other Context | Mechanism (Event / API / Read Model) | Purpose |
|-----------|---------------|--------------------------------------|---------|
| [Outbound / Inbound] | [Context name] | [Mechanism] | [Why] |

## Ownership

| Concern | Owning Context | Owning Team |
|---------|---------------|-------------|
| [Slice of behaviour] | [Context] | [Team] |

## Constraints & Prohibited Actions

1. [Hard constraint — performance, compliance, safety]
2. [Action AI / engineers must not take, e.g., "must not write directly to Equipment DB"]

## Acceptance Criteria (testable)

1. [Given / When / Then — observable behaviour]
2. [Another criterion]

## Open Questions for Domain Expert

| # | Question | Asked Of | Status |
|---|----------|----------|--------|
| 1 | [Ambiguity in PRD] | [Stakeholder] | [Open / Answered] |

## Approval

| Reviewer | Role | Decision | Date |
|----------|------|----------|------|
| [Name] | [Domain expert] | [Approved / Changes requested] | [YYYY-MM-DD] |

## Related Docs

| Doc | Path |
|-----|------|
| Source PRD(s) | [doc/prd.md] |
| Context Map | [../../architecture/context-map.md] |
| Ubiquitous Language | [ubiquitous-language.md] |
| Bounded Context Spec | [bounded-context.md] |
| ADRs | [../../architecture/adr/] |
```

---

## Guidelines

1. **AI drafts, human approves** — never advance to Layer 3 with `Status: Draft`.
2. **One file per bounded context per PRD slice** — if a PRD touches three contexts, produce three intent-translation files.
3. **Use cases are atomic** — one actor, one goal, one primary aggregate. Split if a use case has multiple primaries.
4. **Open Questions are first-class** — if any open question would change ownership or invariants, do not proceed to Layer 3 until it is answered.
5. **Term mapping is mandatory** — every business term used in the use cases must appear in the mapping table or already exist in `ubiquitous-language.md`.
6. **Acceptance criteria must be testable** — if it cannot become a test, rewrite it.
7. **Layer 2 doc** — links upward to PRDs, downward to bounded-context and aggregate docs.
8. **Target ~200 lines** — split by use-case file if it grows larger.
