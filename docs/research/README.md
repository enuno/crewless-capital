# Research

This directory contains Crewless Capital research documents — strategy hypotheses,
framework designs, quant analysis, and conceptual foundations for the firm's autonomous
trading architecture. All documents here are **pre-implementation** artifacts: they inform
system design, agent configuration, and SAE rules but do not constitute live trading
instructions.

---

## Document Registry

| Document | Status | Version | Topics |
|---|---|---|---|
| [Turning "Vibes" into a Composable R&R Framework](./turning-vibes-into-a-composable-framework.md) | Draft | v0 | R currency, PositionSizingService, SAE R-rules, dept R&R profiles, simple vs. adaptive sizing |

---

## Research Standards

- Every document must explicitly separate **facts**, **assumptions**, and **speculation**.
- Strategies are treated as **hypotheses** requiring staged validation — `PAPER_ONLY` before any capital.
- Primary sources preferred: protocol docs, on-chain data, academic literature.
- Frameworks defined here must be **versioned** (`v0`, `v1`, …) before being referenced by SPEC or implementation files.
- No legal, tax, or investment advice.

---

## Related Repositories

- [`enuno/hyperliquid-trading-firm`](https://github.com/enuno/hyperliquid-trading-firm) — Primary implementation SPEC and agent code
- [`enuno/crewless-capital`](https://github.com/enuno/crewless-capital) — Firm constitution, architecture, and research
