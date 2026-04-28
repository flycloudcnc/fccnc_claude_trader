# DDD Template: AI Review Checklist

## Usage

Use this template to produce **`ai-rules/review-checklist.md`** — the checklist a human reviewer (and the AI itself before opening a PR) runs against every AI-assisted change. It operationalizes `development-rules.md` at the PR boundary.

One file per system. Lives at `ai-rules/review-checklist.md`.

---

## Template

```markdown
# AI-Assisted PR Review Checklist

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

> Reviewers: tick every box. An unticked box blocks merge.
> AI agents: run this checklist *before* opening the PR; cite each item in the PR description.

## 1. Intent Traceability

- [ ] PR description names a Use Case ID (`UC-[N]`) from `domains/<ctx>/use-cases.md`.
- [ ] The use case is `Status: Approved` (not `Draft`).
- [ ] Every acceptance criterion in the use case has a corresponding test in this PR.
- [ ] Open Questions on the use case are all `Answered`.

## 2. Domain Boundaries

- [ ] All modified files live under a single `domains/<ctx>/` (or shared `architecture/` / `ai-rules/` with ADR).
- [ ] No imports from another `domains/<other>/` module (HR-1).
- [ ] No new `packages/shared-*` directory (HR-2) — or an ADR is linked.
- [ ] Cross-context interactions go through the published contract; no direct DB or in-process calls into another context.

## 3. Contracts

- [ ] If behaviour visible to other contexts changed, `api-contract.yaml` and/or `events.asyncapi.yaml` were updated in **this** PR (HR-3).
- [ ] OpenAPI / AsyncAPI lints clean (`spectral lint`).
- [ ] No breaking change on an existing event channel (HR-7) — additions only, or new event name with deprecation note.
- [ ] Markdown `integration-contract.md` reflects the YAML diff.
- [ ] Every new event payload field appears in `domain-event-catalog.md`.

## 4. Aggregates & Invariants

- [ ] All writes go through an aggregate root; no external code reaches into child entities.
- [ ] Each invariant in the aggregate doc has a test (HR-4); diff includes test ID -> invariant ID mapping.
- [ ] State-machine transitions in `aggregate.md` match the code's allowed transitions.
- [ ] Concurrency strategy (optimistic locking / version) is preserved.

## 5. Ubiquitous Language

- [ ] All new identifiers use canonical names from `ubiquitous-language.md`.
- [ ] New terms introduced in this PR were added to the glossary.
- [ ] No banned aliases (synonyms list) reintroduced.

## 6. Tests

- [ ] Unit tests for new domain logic.
- [ ] Contract tests against the YAML for any new endpoint or event.
- [ ] Invariant tests for any new or changed invariant.
- [ ] Coverage on changed lines >= the project threshold.

## 7. Governance & ADRs

- [ ] If the change touches ownership, shared kernel, or any HR rule deviation, an ADR is linked (status `Proposed` or `Accepted`).
- [ ] If an ADR is `Proposed`, the PR is marked Draft until accepted.

## 8. AI Audit Trail

- [ ] PR description names the agent / model used.
- [ ] PR description lists the Minimum Input Package items (links accepted) — confirms HR pre-conditions were met.
- [ ] Generated code was reviewed by a human, not auto-merged.
- [ ] No "TODO from AI" left in code without an issue link.

## 9. Non-functional

- [ ] Latency / throughput targets from the use case are still met (load test or rationale).
- [ ] Logging follows the architecture doc's logging convention.
- [ ] No new secret literals; all credentials via the configured secret store.

## 10. Rollout

- [ ] Migration plan documented if the change includes schema migrations.
- [ ] Feature flag or staged rollout if the change is risky and the use case allows.
- [ ] Rollback strategy is at least one sentence.

## Sign-off

| Role | Name | Date |
|------|------|------|
| Code reviewer | | |
| Domain expert (for cross-context changes) | | |
| Architect (for ADR-bearing changes) | | |
```

---

## Guidelines

1. **Every box maps to a hard rule or testable artifact** — vague checks ("looks good") are not allowed.
2. **AI runs the checklist first** — an AI-opened PR with unchecked boxes is auto-rejected by review automation.
3. **Don't grow the list unbounded** — if you find yourself adding a section, ask whether it belongs in `development-rules.md` instead and replace this checklist with a one-line reference.
4. **Sign-off matrix is explicit** — different change types need different reviewers; do not collapse them.
