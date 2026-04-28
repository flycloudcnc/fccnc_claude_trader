# DDD Template: AI Development Rules

## Usage

Use this template to produce **`ai-rules/development-rules.md`** — the binding ruleset every AI-assisted change must respect. This is the operational expression of the PDF's seven governance rules. PRs that violate these are rejected at review.

One file per system. Lives at `ai-rules/development-rules.md`.

---

## Template

```markdown
# AI Development Rules

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]
**Owner:** [Architecture team]

## Hard Rules (violation -> PR rejected)

| # | Rule | Rationale | Verification |
|---|------|-----------|--------------|
| HR-1 | No cross-domain direct database access. All cross-domain reads go through APIs, events, or materialized views. | Prevents hidden coupling and ownership leaks. | Architecture lint: imports of another domain's models/repos fail CI. |
| HR-2 | No "shared common business model" packages. Shared kernels require an explicit ADR and are limited to value objects with no behaviour. | Shared kernels become god-modules. | Build-time check: `packages/shared-*` is allowlisted only if an ADR file exists. |
| HR-3 | Contracts (`api-contract.yaml`, `events.asyncapi.yaml`) and implementations are updated in the same PR. | Prevents "spec drift". | CI fails if implementation files in `domains/<ctx>/` change without a contract diff (or an ADR explaining why). |
| HR-4 | Aggregate invariants documented in `aggregate.md` must have at least one matching test. | Invariants without tests are aspirational. | Coverage check maps each invariant ID to a test ID. |
| HR-5 | Ambiguous ownership requires an ADR before code is written. | Forces explicit decisions. | PR template requires ADR link when the "ambiguous-ownership" label is set. |
| HR-6 | Every change must map to at least one acceptance criterion from a use case. | Ties code back to business intent. | PR description must reference `UC-[N]`. |
| HR-7 | Domain events are append-only — schema breaking changes require a new event name and a deprecation window. | Protects subscribers. | AsyncAPI diff lint blocks breaking changes on existing channels. |

## AI Scope Rules (what the AI agent may and may not change)

| Allowed | Not Allowed (without explicit human approval) |
|---------|----------------------------------------------|
| Implement use cases inside one bounded context | Modify another context's aggregate |
| Add/modify code under `domains/<ctx>/` if the contract is updated in the same PR | Edit a published `api-contract.yaml` to remove or rename a field |
| Add tests for invariants and acceptance criteria | Delete or weaken an invariant test |
| Update the markdown contract to reflect a YAML contract change | Introduce a new cross-context dependency without an integration-contract entry |
| Refactor within an aggregate | Move code across bounded-context boundaries |
| Update `use-cases.md` open questions | Mark a use case `Approved` (only domain experts may) |

## Minimum Input Package

An AI agent must not start a code change unless **all** of these are present in the prompt or referenced files:

1. Bounded context name and link to `bounded-context.md`
2. Business goal (one sentence) from the source use case
3. Use case ID (`UC-[N]`) and link to `use-cases.md`
4. Domain invariants the change must preserve
5. API / event contract reference (`api-contract.yaml` or event names)
6. Acceptance criteria (testable Given/When/Then)
7. Impacted repos / modules
8. Non-functional requirements (latency, throughput, security)
9. Prohibited actions (from this rules file plus any use-case-specific constraints)

If any item is missing, the AI must stop and request it instead of guessing.

## Repository Boundaries

| Path | Owner | AI may modify? |
|------|-------|----------------|
| `architecture/` | Architecture team | Only via ADR proposal |
| `architecture/adr/` | Architecture team | May propose new ADRs (status: Proposed) |
| `domains/<ctx>/` | Context's owning team | Yes, within HR-1..HR-7 |
| `ai-rules/` | Architecture team | No — propose changes via ADR |
| `services/`, `apps/`, `packages/` | Per-service `CODEOWNERS` | Yes, within HR-1..HR-7 |
| `tests/` | Owning team | Yes |

## Escalation

- **Conflict between two contexts** -> open ADR, status `Proposed`.
- **Governance rule appears to block a legitimate need** -> ADR with `Governance Impact: Deviates` and a compensating control.
- **AI cannot identify an owner** -> stop; do not write code; surface to the architect.

## Related Docs

| Doc | Path |
|-----|------|
| Review Checklist | [review-checklist.md] |
| Workflow Templates | [workflow-templates.md] |
| Context Map | [../architecture/context-map.md] |
| ADR Index | [../architecture/adr/] |
```

---

## Guidelines

1. **Hard rules are tested by CI** — every HR-N must be enforced automatically; a rule that depends only on reviewer goodwill is decoration.
2. **Keep the rule list short** — if the system has more than ~10 hard rules, the model is too coarse; push detail into context-specific rules inside `domains/<ctx>/`.
3. **The Minimum Input Package is non-negotiable** — it is what turns "prompting" into "engineering" (PDF wording).
4. **AI Scope Rules name the file paths** — vagueness defeats the purpose.
5. **Versioned** — every change ships with a bump and an ADR.
