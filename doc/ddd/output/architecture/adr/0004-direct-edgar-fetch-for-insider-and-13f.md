# ADR-0004: Direct SEC EDGAR Fetch for Insider Form 4 and 13F-HR Filings

**Status:** Proposed
**Date:** 2026-04-28
**Deciders:** Architecture team
**Consulted:** Owner of `market-data` BC; consumer owners (`alpha-generation`, `catalyst-timing`)
**Informed:** Implementers, compliance reviewer

## Context

PRD §6 lists `insider transactions` and `13F filings` as required data, and the investment-frameworks document calls them out heavily (frameworks §11 and the Burry / catalyst frameworks). Two practical sources exist:

1. **A commercial vendor** — clean schema, push API, but a paid subscription with potentially restrictive redistribution terms.
2. **SEC EDGAR directly** — authoritative source of Form 4 (insider transactions) and 13F-HR (institutional holdings). Free, no per-call cost, but raw filings are XBRL/SGML and require parsing; EDGAR mandates a polite User-Agent and rate-limits at 10 req/s. There is also a 45-day delay on 13F-HR filings inherent to the SEC requirement, regardless of source.

The `catalyst-timing` BC needs cluster detection (`3+ insiders in 30 days`) which means we need raw transaction-level data, not vendor summaries.

## Decision

We will fetch insider Form 4 and 13F-HR filings **directly from SEC EDGAR**, parse them in an Anti-Corruption Layer inside `market-data`, and emit `InsiderTransactionRecorded` and `ThirteenFFilingRecorded` events. We will **not** use a commercial insider/13F vendor for v1. The ACL is responsible for translating EDGAR's filing structures into the `InsiderTransaction` and `ThirteenFFiling` aggregates as defined in `domains/market-data/`.

## Scope

| Aspect | Value |
|--------|-------|
| Bounded Context(s) affected | `market-data` (publisher); `alpha-generation`, `catalyst-timing` (consumers) |
| Aggregates affected | `InsiderTransaction`, `ThirteenFFiling` |
| Contracts affected | `architecture/contracts/market-data-alpha-generation.md`, `architecture/contracts/market-data-catalyst-timing.md`; `domains/market-data/events.asyncapi.yaml` |
| Code modules affected | `src/signal_intelligence/market_data/adapters/edgar/*` |

## Options Considered

| # | Option | Pros | Cons |
|---|--------|------|------|
| 1 | Commercial vendor (e.g. WhaleWisdom, InsiderArbitrage) | Clean schema; push API; less code | Per-month subscription; redistribution license risk; vendor-shaped models leak unless we still write an ACL; sometimes lossy aggregations |
| 2 | **SEC EDGAR directly with an ACL (Chosen)** | Authoritative; free; no redistribution issue; raw data preserves cluster-detection capability | Must parse XBRL/SGML; must respect EDGAR rate limits; more code in the adapter |
| 3 | Both (vendor primary, EDGAR fallback) | Resilience | Doubles maintenance for v1; not justified by single-account v1 reliability needs |

## Consequences

**Positive**
- Authoritative source — no vendor schema drift.
- Zero subscription cost; aligns with single-operator self-hosted topology.
- Raw transactions enable `catalyst-timing` cluster detection without re-aggregating vendor summaries.
- ACL keeps EDGAR field naming out of the domain model — future swap to a vendor is a port substitution.

**Negative**
- We own the parser. EDGAR XBRL is well-documented but verbose; expect occasional filer-format quirks.
- Rate-limit discipline is on us — must throttle to ≤10 req/s and use a descriptive User-Agent (SEC requires this).
- 13F filings have an inherent 45-day reporting delay (SEC rule); document this in `event-catalog.md` so consumers don't expect intra-day signals from `ThirteenFFilingRecorded`.

**Neutral / Follow-up**
- Implement the EDGAR fetcher as a thin adapter behind an `InsiderFilingFetcher` port.
- Write contract tests against real archived filings (3-5 fixtures across normal + edge formats).
- Add a "vendor fallback" ADR if EDGAR availability turns out to be unreliable in production.

## Governance Impact

| Governance Rule | Effect of this ADR |
|-----------------|--------------------|
| HR-1 (no cross-context DB reads) | Conforms — EDGAR is an external source, not another BC's DB. |
| External-source ACL requirement (per `architecture.md`) | Conforms — `market-data` owns the ACL. |
| HR-3 (contracts linted in CI) | Conforms — emitted events are defined in `events.asyncapi.yaml` regardless of source. |

No deviation.

## Verification

- The EDGAR adapter unit-tests against fixture filings (Form 4 standard + Form 4 amended + 13F-HR with footnote schedule).
- A contract test asserts `InsiderTransactionRecorded` payload matches the AsyncAPI schema exactly when parsing fixture filings.
- A rate-limit unit test verifies the adapter never issues more than 10 req/s and always sets a non-empty User-Agent.
- CI fails if any code outside `src/signal_intelligence/market_data/adapters/edgar/` references EDGAR field names directly.

## Related Docs

| Doc | Path |
|-----|------|
| Context Map (ACL row for EDGAR) | [../context-map.md](../context-map.md) |
| Architecture (External APIs row) | [../architecture.md](../architecture.md) |
| Event Catalog (`InsiderTransactionRecorded`, `ThirteenFFilingRecorded`) | [../event-catalog.md](../event-catalog.md) |
| Contract: market-data → catalyst-timing | [../contracts/market-data-catalyst-timing.md](../contracts/market-data-catalyst-timing.md) |
| Development Rules | [../../ai-rules/development-rules.md](../../ai-rules/development-rules.md) |
