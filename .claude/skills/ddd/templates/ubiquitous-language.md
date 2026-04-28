# DDD Template: Ubiquitous Language Glossary

## Usage

Use this template to maintain the **canonical vocabulary** for the entire system. This ensures consistent naming across code, documentation, conversations, and AI-generated output. When a term has different meanings in different bounded contexts, document each meaning separately.

---

## Template

```markdown
# Ubiquitous Language Glossary

**Version:** [X.Y]
**Last Updated:** [YYYY-MM-DD]

## System-Wide Terms

| Term | Definition | Code Name | Example |
|------|-----------|-----------|---------|
| [Domain Term] | [Precise definition] | [How it appears in code: class/variable/function name] | [Concrete example] |

## Context-Specific Terms

### [Bounded Context Name]

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| [Term] | [What it means here] | [Code representation] | [How other contexts define this term, if different] |

### [Another Bounded Context]

| Term | Definition in This Context | Code Name | Differs From |
|------|---------------------------|-----------|-------------|
| [Term] | [What it means here] | [Code representation] | [Differences from other contexts] |

## Synonyms and Aliases

| Preferred Term | Aliases (Do Not Use) | Reason |
|---------------|---------------------|--------|
| [Canonical name] | [Other names people might use] | [Why we chose this term] |

## Naming Conventions

| Category | Convention | Example |
|----------|-----------|---------|
| **Entities** | [PascalCase noun] | `Trade`, `Position` |
| **Value Objects** | [PascalCase noun/adjective] | `Price`, `Quantity` |
| **Events** | [PascalCase past tense] | `TradeExecuted` |
| **Commands** | [PascalCase imperative] | `ExecuteTrade` |
| **Services** | [PascalCase noun + Service] | `RiskCalculationService` |
| **Repositories** | [PascalCase noun + Repository] | `TradeRepository` |

## Ambiguous Terms to Avoid

| Term | Problem | Use Instead |
|------|---------|-------------|
| [Ambiguous term] | [Why it causes confusion] | [Preferred alternative(s)] |

## Related Docs

| Doc | Path |
|-----|------|
| Context Map | [context-map.md] |
| [Context A] Spec | [contexts/<context-a>.md] — context-specific terms |
| [Context B] Spec | [contexts/<context-b>.md] — context-specific terms |
```

---

## Guidelines

1. **One glossary per system** — with context-specific subsections where meanings diverge.
2. **Code names must match** — if the glossary says "Trade", the code must use `Trade`, not `Order` or `Deal`.
3. **AI agents must use these terms** — when generating code, comments, or docs, use the canonical names.
4. **Update on every model change** — new entities, renamed concepts, or retired terms must be reflected here.
5. **Flag ambiguity explicitly** — if a term means different things in different contexts, document both meanings.
6. **Synonyms are banned** — pick one term and redirect all aliases to it.
7. **Layer 1 doc** — load alongside the context map when starting any task. This ensures consistent terminology from the start.
