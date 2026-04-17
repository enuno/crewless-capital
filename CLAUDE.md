# CLAUDE.md — Crewless Capital

Autonomous, self-custodied, DEX-native AI trading firm. Multi-department
architecture where LLM agents produce typed analytical artifacts and
deterministic services enforce all risk, wallet, and execution authority.

Read SPEC.md and AGENTS.md before making any changes.

---

## Constitutional Invariants — Never Violate
1. LLM output is never executable trading authority. LLMs produce typed reports,
   theses, and intents. `firm-risk-core`, `wallet-policy-service`, and
   `policy-engine` enforce all execution authority — deterministically.
2. Never bypass or add conditionals to `firm-risk-core`, `wallet-policy-service`,
   or `policy-engine`. No `if dev_mode`, no feature flags, no exceptions.
3. All inter-service handoffs use typed, schema-validated protobuf/JSON artifacts.
   No raw strings, untyped dicts, or prompt-chained state crosses service
   boundaries. Invalid artifacts trigger failover or cycle abort — never silent
   coercion.
4. All adaptation (strategy changes, prompt updates, optimizer outputs) goes
   through versioned promotion gates. Never mutate live behavior mid-session in
   `FirmMode.LIVE`.
5. Every cycle output is persisted to `decision_traces` keyed on `cycle_id`.
   Traces are append-only — never update or delete trace records.
6. `HOLD_FLAT` is always a valid first-class return value. Never coerce an agent
   toward a trade. In `LIVE` mode, if no provider meets minimum model tier,
   emit `HOLD_FLAT` and log `policy_reject` — do not execute on a degraded model.
7. On-chain session key scope (ERC-4337/ERC-7579) is the enforcement layer for
   wallet authority — not the database alone. Database grants must mirror
   on-chain constraints.
8. Environment separation is absolute. Research, paper, shadow, and live must not
   share data flows without an explicit promotion gate event in `governance_events`.

---

## Language and Runtime Defaults
- **Python** (3.11+): agent pipelines, strategy, research, optimizer jobs.
  Pydantic v2 for all artifact validation. `structlog` for structured JSON logs.
  `pytest` + `pytest-asyncio` for tests. Never use bare `print()`.
- **TypeScript** (`strict: true`, no `any`): orchestrator-api, firm-risk-core,
  wallet-policy-service, dashboard. Zod for runtime validation. `pino` for logs.
- **Protos** are the source of truth for all contracts → `proto/`.
  Run `make proto-gen` after any change. Never hand-edit `packages/schemas/`.
- **Chain adapters**: Python primary; Rust only for proven latency-critical paths.

---

## CIO Agent — Two-Stage Contract (High Stakes)
Stage 1: Policy Engine deterministic pre-filter removes ineligible departments
(PAUSED / SUNSETTING / UNFUNDED / circuit breaker / reconciliation drift /
stale_data_incident in last 2 cycles / operational_health_score < 0.70).
The model never sees pre-filtered departments and cannot override this stage.
Stage 2: Model receives structured `AllocationScoringInput`; returns typed
`AllocationRationale`. Budget changes without numeric justification from the
input must be flagged `UNSUBSTANTIATED` and are auto-rejected.
Stage 3: Policy Engine post-check — CRO caps win unconditionally over CIO output.
Never merge code that allows the CIO model to skip Stage 1 or Stage 3.

---

## Execution Authority Stack (must be traversed in order, no steps skipped)
Department budget check → Firm risk engine → Wallet authority grant →
Protocol allowlist → On-chain session key → Transaction builder →
Signer → Broadcast → Fill reconciliation → Immutable trace append

---

## Storage Rules
- Hard trading state (positions, fills, PnL, wallet balances): **Postgres only**.
  Never store in Supermemory.
- Agent semantic memory (reflections, regime narratives, debate summaries):
  **Supermemory.ai** per-agent `containerTag`.
- Time-series market data, OHLCV, oracle prices: **TimescaleDB**.
- Redis / NATS: transient coordination only — never for audit state.

---

## Testing — Non-Negotiable
- Every deterministic safety gate needs a negative-path test asserting it cannot
  be bypassed — not just a happy-path test.
- PRs touching `firm-risk-core` or `wallet-policy-service` must pass
  `tests/chaos/` before merge.
- Every typed artifact producer must assert schema validity on all output paths,
  including error paths.

---

## Key File Pointers
- Typed contracts → `proto/`
- Prompt policies (versioned; promotion gate required for LIVE changes)
  → `packages/prompt-policies/`
- Model routing table (primary / failover / live-mode minimums)
  → `packages/model-routing/`
- Wallet authority rule definitions → `packages/wallet-policies/`
- Firm policy YAML → `config/policies/`
- Bridge allowlist and risk parameters → `config/bridges/`
- Per-department configuration and mandates → `config/departments/`
- New component checklists (department, adapter, bridge, prompt policy)
  → `AGENTS.md §17`
