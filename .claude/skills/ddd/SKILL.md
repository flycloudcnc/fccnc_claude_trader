---
name: ddd
description: Translate one or more PRD/design docs (passed as arguments to `/ddd`) into Governed AI Software Delivery documentation (six layers, by-domain layout) using the templates bundled at `.claude/skills/ddd/templates/`. Use when the user runs `/ddd <path> [<path>...]`, asks to model the system as DDD, or wants to (re)generate the use cases, bounded contexts, aggregates, events, contracts, ADRs, or AI rules from the PRD.
---

# /ddd — PRD -> Governed AI Delivery Docs

Turn one or more PRD/design docs into a layered set of governance documents covering Layers 2-4 and 6 of the Governed AI Software Delivery model (PDF: `doc/governed_ai_software_delivery.pdf`). Layer 1 (PRD) is the input; Layer 5 (implementation) is handled by `/implement`. Templates ship with this skill at `.claude/skills/ddd/templates/`. Outputs go to a **by-domain layout**:

- `architecture/` — system-wide docs (context map, architecture, glossary, event catalog, ADRs, markdown contracts)
- `domains/<context>/` — per-context docs (use cases, bounded context, aggregates, OpenAPI/AsyncAPI specs)
- `ai-rules/` — governance rules, review checklist, workflow templates

Run from the repo root.

## Invocation

```
/ddd <path-or-glob> [<path-or-glob> ...]
```

The PRD may be split across multiple files. Examples:

```
/ddd doc/prd.md
/ddd doc/prd.md doc/investment-frameworks.md
/ddd doc/*.md
```

If the user invokes `/ddd` with **no arguments**, stop and ask which PRD/design files to use — do **not** assume `doc/prd.md`. Resolve every argument to absolute paths inside the repo before reading.

## Inputs

- **Required:** every file resolved from the invocation arguments — read **all** of them and treat them collectively as the source of truth for scope, agents, data, and rules.
  - If any argument resolves to zero files, stop and report the bad path.
  - If two input files disagree, surface the conflict in the proposal step (step 3) and ask the user which wins. Do not silently pick one.
- **Required (bundled):** `.claude/skills/ddd/templates/*.md` — canonical templates ship with this skill. Follow each template **exactly** (sections, tables, headings). Do not look elsewhere for templates; do not modify these files.
- **Reference:** `.claude/skills/ddd/templates/README.md` — explains the layered model, by-domain layout, generation order, and loading strategy. Read it before starting.

## Bundled Templates (by layer)

| Template | Layer | Output path | Purpose |
|----------|-------|-------------|---------|
| `intent-translation.md` | 2 | `domains/<ctx>/use-cases.md` | Bridge business intent to domain spec |
| `context-map.md` | 3 | `architecture/context-map.md` | System-wide context overview |
| `architecture.md` | 3 | `architecture/architecture.md` | Cross-cutting technical decisions |
| `ubiquitous-language.md` | 3 | `architecture/ubiquitous-language.md` (+ `domains/<ctx>/ubiquitous-language.md` for overrides) | Canonical vocabulary |
| `domain-event-catalog.md` | 3-4 | `architecture/event-catalog.md` | All domain events |
| `bounded-context.md` | 3 | `domains/<ctx>/bounded-context.md` | One bounded context spec |
| `aggregate.md` | 3 | `domains/<ctx>/aggregates/<aggregate>.md` | One aggregate deep-dive |
| `api-contract.md` | 4 | `domains/<ctx>/api-contract.yaml`, `domains/<ctx>/events.asyncapi.yaml` | Machine-readable OpenAPI 3.1 / AsyncAPI 2.6 |
| `integration-contract.md` | 4 | `architecture/contracts/<a>-<b>.md` | Markdown contract describing the integration intent |
| `adr.md` | 6 | `architecture/adr/NNNN-<slug>.md` | Architecture Decision Record |
| `ai-development-rules.md` | 6 | `ai-rules/development-rules.md` | Hard rules + AI scope + minimum input package |
| `ai-review-checklist.md` | 6 | `ai-rules/review-checklist.md` | PR review checklist |
| `ai-workflow-templates.md` | 6 | `ai-rules/workflow-templates.md` | Canonical AI prompt templates |

## Output layout (by-domain)

```
project-root/
 ├── architecture/
 │    ├── context-map.md
 │    ├── architecture.md
 │    ├── ubiquitous-language.md
 │    ├── event-catalog.md
 │    ├── adr/NNNN-<slug>.md
 │    └── contracts/<a>-<b>.md
 ├── domains/
 │    └── <context>/
 │         ├── use-cases.md
 │         ├── ubiquitous-language.md          (overrides only)
 │         ├── bounded-context.md
 │         ├── invariants.md                    (cross-aggregate rules)
 │         ├── aggregates/<aggregate>.md
 │         ├── api-contract.yaml
 │         └── events.asyncapi.yaml
 └── ai-rules/
      ├── development-rules.md
      ├── review-checklist.md
      └── workflow-templates.md
```

For this repo specifically, `architecture/`, `domains/`, and `ai-rules/` live under `doc/ddd/output/` (the repo's existing scratch area). When generating, write under `doc/ddd/output/<architecture|domains|ai-rules>/...`. Never write outside `doc/ddd/output/`.

If `doc/ddd/output/` does not exist, create it. Never overwrite a file silently.

## Procedure

1. **Read inputs.** Read every PRD/argument end-to-end and `.claude/skills/ddd/templates/README.md`. Skim the other templates so you know what each one expects.

2. **Inventory existing output.** List `doc/ddd/output/` and note what already exists. Do not overwrite a file silently — call it out in the proposal and ask before replacing.

3. **Propose the model.** Before generating anything, present a short plan to the user:
   - Bounded contexts you intend to define (name + 1-line purpose) — derived from the PRDs' agents, data, and execution loop.
   - Aggregates you will create per context (with the aggregate root).
   - Domain events you will catalog.
   - Integration contracts (upstream -> downstream pairs and style: sync / async / CQRS / shared kernel).
   - Use cases per context (UC-N labels) and any open questions for the domain expert.
   - ADRs you anticipate needing (typically: ambiguous ownership, shared kernel proposal, breaking contract change).
   - Whether `ai-rules/*.md` should be (re)generated in this run.
   - Which files are NEW vs. would OVERWRITE an existing output file.
   - **Stop and confirm with the user before writing any docs.**

4. **Generate top-down by layer.** After confirmation, generate in this order. For each file, fill the matching template literally — same sections, same tables, same headings — and respect the template's stated max-line budget.

   1. `architecture/context-map.md`
   2. `architecture/architecture.md`
   3. `architecture/ubiquitous-language.md`
   4. `architecture/event-catalog.md`
   5. `domains/<ctx>/use-cases.md` — Layer 2, **Status: Draft (AI)**; warn the user that domain-expert approval is required before Layer 3 generation per `intent-translation.md` rules
   6. `domains/<ctx>/bounded-context.md` — one per bounded context
   7. `domains/<ctx>/aggregates/<aggregate>.md` — one per aggregate
   8. `domains/<ctx>/api-contract.yaml` and `domains/<ctx>/events.asyncapi.yaml` — Layer 4 wire format
   9. `architecture/contracts/<upstream>-<downstream>.md` — Layer 4 markdown counterpart
   10. `architecture/adr/NNNN-<slug>.md` — only when a Layer 2 open question or Layer 3 conflict requires one
   11. `ai-rules/development-rules.md`, `review-checklist.md`, `workflow-templates.md` — only on the first run, or when the user explicitly asks to refresh

   Every doc must include a `Related Docs` section with correct relative links so the layered docs stay navigable.

5. **Cross-check.** After generating, verify:
   - Every context listed in `context-map.md` has matching `domains/<ctx>/bounded-context.md` and `use-cases.md`.
   - Every aggregate listed in a context doc has a matching `aggregates/<aggregate>.md`.
   - Every event listed in a context or aggregate appears in `event-catalog.md` and in the publishing context's `events.asyncapi.yaml`.
   - Every relationship in `context-map.md` has matching `architecture/contracts/<upstream>-<downstream>.md` (where applicable).
   - Every command in `bounded-context.md` has an OpenAPI `operationId` in the context's `api-contract.yaml` (where applicable).
   - Every invariant in an aggregate has a corresponding acceptance criterion in `use-cases.md` (or an explicit note that it is implementation-internal).
   - Ubiquitous-language terms used in context docs are defined in `architecture/ubiquitous-language.md` (or the per-context override).
   - Any `Shared Kernel` relationship in `context-map.md` is backed by an ADR (HR-2).

6. **Report.** Print:
   - Tree of files created/updated under `doc/ddd/output/`.
   - Any PRD areas intentionally not modelled and why.
   - Any open questions where the PRD was ambiguous (so the user can clarify in a follow-up pass) — surfaced as Open Questions in the affected `use-cases.md` files.
   - List of use-case files still in `Status: Draft (AI)` — Layer 3 docs derived from those should be treated as provisional until approved.

## Context-window discipline

The DDD pass can be large. Follow the templates README's rule: **commit, push, and suggest a fresh session before context reaches ~50%.**

- The four `architecture/*.md` Layer 3 docs are usually safe in one session.
- Generate `use-cases.md` (Layer 2) per context in its own session — domain-expert review happens between sessions.
- Generate Layer 3 contexts in one session each, or 2–3 small ones together.
- Generate Layer 3 aggregates and Layer 4 contracts (markdown + YAML pair) **one per session**.
- ADRs and `ai-rules/*.md` each get their own session.
- After each meaningful batch, commit and push, then stop and tell the user to start a new conversation to continue.

## Rules

- **Never write any output file before the user confirms the proposal in step 3.**
- Follow each template exactly — do not invent new sections or drop required ones.
- Respect the max-line budgets given in each template (see `templates/README.md`).
- Do not duplicate definitions: events live in `event-catalog.md` + AsyncAPI; terms in `ubiquitous-language.md`; entity detail in aggregate docs; wire format in YAML. Other docs reference by name.
- Do not modify files outside `doc/ddd/output/`. In particular, never modify `CLAUDE.md`, the templates, or the PRD.
- This is a documentation workflow only. Do not write or change application code. Implementation belongs to `/implement` (Layer 5).
- Layer 2 (`use-cases.md`) outputs always start at `Status: Draft (AI)`. Mark them as such and tell the user explicitly that a domain expert must approve before Layer 3 docs derived from them are trusted.
- A `Shared Kernel` relationship or any breaking-contract change requires a draft ADR in the same run (status `Proposed`) — surface this in the proposal step.
