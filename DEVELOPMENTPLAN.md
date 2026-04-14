# Crewless Capital — Development Plan

**Version:** 2.0.0-draft  
**Last Updated:** 2026-04-13  
**Status:** Active development  
**Spec Reference:** [SPEC.md](SPEC.md)  
**Repo:** https://github.com/enuno/crewless-capital

---

## Guiding Rules

Before reading the phases, these rules govern the entire development process:

1. **Contract-first.** Protobuf definitions and JSON schemas are written before any service implementation. Services are built to fulfill contracts, not the other way around.
2. **Exit gates are hard.** No phase begins until the previous phase's exit gate is fully satisfied and verified. Gate criteria are not negotiable.
3. **Paper before shadow, shadow before live.** No capital is deployed until the system has operated correctly in paper mode, then shadow mode, for a defined period with verified trace quality.
4. **Safety services before execution services.** The firm risk core, wallet policy service, and policy engine are implemented and tested before any execution adapter is wired.
5. **Research and optimization are always off the hot path.** No phase introduces a mechanism by which research or optimizer output can mutate live behavior without a versioned promotion gate.
6. **Every phase produces a measurable artifact.** Progress is not tracked by lines of code — it is tracked by verifiable outputs: passing contract tests, complete traces, measured metrics, and explicit gate sign-offs.
7. **Minimal viable scope first.** Phases 0–5 target only the three initial departments: Momentum, Market Neutral, and Treasury & Yield. Additional departments are added in Phase 6+.

---

## Phase Overview

| Phase | Name | Status | Gate |
|---|---|---|---|
| 0 | Foundation & Contracts | 🔲 Not started | All services boot; contract tests pass |
| 1 | Data & Research Layer | 🔲 Not started | Complete research packets for 3 assets |
| 2 | Department Decision Pipeline | 🔲 Not started | Full traces: debate + intent for 3 departments |
| 3 | Allocation Engine & Executive Layer | 🔲 Not started | Allocation decisions logged; department states transition |
| 4 | Safety Layer & Wallet Authority | 🔲 Not started | No execution reaches adapter without firm decision + grant |
| 5 | Paper Trading & Evaluation Harness | 🔲 Not started | Continuous paper cycles with full traces and metrics |
| 6 | Shadow Allocation & Comparison | 🔲 Not started | Multi-dept shadow vs. single-strategy baseline measured |
| 7 | Treasury & Yield Live (Limited) | 🔲 Not started | Treasury dept live at hard cap; all gates validated |
| 8 | Staged Live Expansion | 🔲 Not started | Momentum and Market Neutral promoted through defined gates |
| 9 | Department Expansion | 🔲 Not started | Additional departments onboarded via department scaffold |

---

## Phase 0 — Foundation & Contracts

**Goal:** Establish the repo scaffold, protobuf contract definitions, codegen pipeline, local development environment, and health endpoints for all planned services. Nothing functional is built in this phase — only the skeleton on which everything else is built.

### Deliverables

- [ ] Repo directory structure created per `SPEC.md §9`
- [ ] All `.proto` files authored with complete message and enum definitions per `SPEC.md §8`
  - `common.proto` — FirmMeta, FirmMode, shared enums
  - `market.proto` — ObservationPack, MarketSnapshot, PriceData
  - `department.proto` — DepartmentStatus, DepartmentIntentSet, DepartmentIntent, DepartmentState
  - `allocation.proto` — CapitalAllocationDecision, DepartmentBudget
  - `risk.proto` — RiskVote, RiskReview, RiskVerdict, FirmExecutionDecision
  - `wallet.proto` — WalletAuthorityGrant, WalletClass, WalletBalance
  - `execution.proto` — ExecutionPlan, OnchainAction, FillReport, DecisionTrace
  - `treasury.proto` — TreasuryEvent, ReserveSnapshot, YieldDeployment
  - `governance.proto` — GovernanceEvent, HITLRuleSet, HumanApproval, PromotionEvent
  - `research.proto` — ResearchPacket, AnalystReport, DebateOutcome, TradeThesis
- [ ] JSON schemas generated from proto definitions (`packages/schemas/`)
- [ ] Proto codegen pipeline configured (Python + TypeScript targets)
- [ ] `docker-compose.yml` — Postgres, Redis/NATS, MLflow, all service stubs
- [ ] `docker-compose.paper.yml` — paper trading environment overlay
- [ ] `.env.example` with all required env vars documented
- [ ] `Makefile` targets: `proto`, `codegen`, `test-contracts`, `up`, `down`, `logs`
- [ ] Health endpoint (`GET /health`) implemented on all service stubs
- [ ] Contract test suite scaffolded (`tests/contract/`)
- [ ] All services boot without errors in docker-compose

### Exit Gate

```
✅ All service stubs start and return 200 on GET /health
✅ Proto codegen pipeline produces valid Python and TypeScript client libraries
✅ JSON schemas validate against all proto message shapes
✅ Contract tests pass: all required message fields present, enums valid
✅ docker-compose up brings full stack to healthy state
```

### Service targets this phase

| Service | Output |
|---|---|
| All services | Stub with /health endpoint |
| proto/ | All .proto files complete |
| packages/schemas/ | Generated JSON schemas |
| docker-compose | Full local stack |

---

## Phase 1 — Data & Research Layer

**Goal:** Implement the market and on-chain data ingestion pipeline and the department analyst agents. At the end of this phase, the system can produce a complete, typed `ResearchPacket` for any target asset without any trading decision being made.

### Deliverables

- [ ] `apps/research-bridge/` — market data ingestion service
  - [ ] OHLCV data ingestion (configurable source: on-chain oracle or feed adapter)
  - [ ] Order book / liquidity depth ingestion
  - [ ] Funding rate ingestion (where applicable by protocol)
  - [ ] On-chain signal ingestion: TVL, protocol flows, token transfers, governance activity
  - [ ] News and narrative feed adapter
  - [ ] `ObservationPack` emitted on schedule to event bus
- [ ] Department analyst agents (within `departments/runtime-host/` or per-department)
  - [ ] Technical Analyst — price action, volume, momentum indicators
  - [ ] On-Chain Analyst — flow, protocol activity, whale behavior
  - [ ] Flow / Liquidity Analyst — order book depth, slippage conditions, liquidity regime
  - [ ] Protocol / Governance Analyst — governance events, vesting, emissions
  - [ ] News / Narrative Analyst — sentiment, macro narrative, event calendar
- [ ] `ResearchPacket` typed output per analyst, per cycle, per department
- [ ] Analyst outputs persisted to Postgres (`analyst_reports` table)
- [ ] `ObservationPack` and `ResearchPacket` schema validation on every output
- [ ] Model routing table implemented (`packages/model-routing/`)
  - Fast/cheap model for data normalization and entity extraction
  - Mid-tier model for analyst synthesis
- [ ] `apps/research-bridge/` integration tests

### Exit Gate

```
✅ System produces a complete ResearchPacket for BTC, ETH, and one additional asset
✅ All 5 analyst agents emit typed AnalystReport outputs with no schema violations
✅ ObservationPack and ResearchPacket are persisted to Postgres with correct cycle_id
✅ No analyst output contains executable trading intent (reports only)
✅ Model routing correctly dispatches by task type
```

### Service targets this phase

| Service | Output |
|---|---|
| `apps/research-bridge/` | Full data ingestion + analyst pipeline |
| `packages/model-routing/` | LLM task routing table |
| Postgres | `analyst_reports`, `observation_packs` tables live |

---

## Phase 2 — Department Decision Pipeline

**Goal:** Implement the full intra-department decision pipeline: bull/bear debate, trader synthesis, department risk agent, and portfolio agent. At the end of this phase, the system produces complete `DepartmentIntentSet` and `DepartmentStatus` artifacts for each of the three initial departments, with full trace persistence. No execution occurs.

### Deliverables

- [ ] `departments/runtime-host/` — generic department pipeline host
  - [ ] Configurable per-department via `mandate.yaml`
  - [ ] Accepts `ObservationPack` + `ResearchPacket` as inputs
  - [ ] Emits `DepartmentStatus` and `DepartmentIntentSet` as outputs
- [ ] Bull researcher agent — constructs thesis from ResearchPacket
- [ ] Bear researcher agent — constructs counter-thesis
- [ ] Debate facilitator — runs N-round adversarial debate, emits `DebateOutcome`
- [ ] Trader / Strategy Synthesizer — produces `DepartmentIntent` proposals from DebateOutcome
- [ ] Department Risk Agent — evaluates intent proposals, emits local risk assessment
- [ ] Department Portfolio Agent — applies mandate constraints, emits `DepartmentIntentSet`
- [ ] `mandate.yaml` definitions for three initial departments:
  - [ ] `departments/momentum/mandate.yaml`
  - [ ] `departments/market-neutral/mandate.yaml`
  - [ ] `departments/treasury-yield/mandate.yaml`
- [ ] Prompt policies authored for all department agents (`packages/prompt-policies/`)
- [ ] `DepartmentStatus` populated with rolling metrics (zeroed initially, accumulated over cycles)
- [ ] Department pipeline integration tests
- [ ] `apps/orchestrator-api/` — cycle trigger endpoint (`POST /cycles/trigger`)
- [ ] `DecisionTrace` partially populated through department stage, persisted to Postgres

### Exit Gate

```
✅ POST /cycles/trigger returns a complete DecisionTrace with:
   - ResearchPacket for all 3 departments
   - DebateOutcome for all 3 departments
   - DepartmentIntentSet for all 3 departments
   - DepartmentStatus for all 3 departments
✅ All artifacts schema-valid with no missing required fields
✅ No DepartmentIntent contains a wallet_id or grant_id (execution not yet wired)
✅ HOLD_FLAT is correctly emitted when debate consensus is below threshold
✅ Traces persisted to Postgres and queryable by cycle_id
✅ Strong reasoning model correctly used for debate and trader synthesis stages
```

### Service targets this phase

| Service | Output |
|---|---|
| `departments/runtime-host/` | Generic pipeline host |
| `departments/momentum/` | Mandate + agents |
| `departments/market-neutral/` | Mandate + agents |
| `departments/treasury-yield/` | Mandate + agents |
| `apps/orchestrator-api/` | Cycle trigger + trace store |
| `packages/prompt-policies/` | All department agent prompts versioned |
| Postgres | `decision_traces`, `department_intents`, `debate_outcomes` tables live |

---

## Phase 3 — Allocation Engine & Executive Layer

**Goal:** Implement the Executive Layer — CIO allocator, CRO risk aggregation, and the allocation scoring engine. At the end of this phase, the system produces a `CapitalAllocationDecision` each cycle that shifts virtual capital across departments based on performance, regime, and strategic priority. No live execution yet.

### Deliverables

- [ ] `apps/allocation-engine/` — executive scoring and budget routing service
  - [ ] `DepartmentStatus` ingestion from all active departments
  - [ ] Regime classification (trending / ranging / volatile / defensive)
  - [ ] Allocation scoring per department across all dimensions in `SPEC.md §5.2`
  - [ ] `CapitalAllocationDecision` emitted per allocation cycle
  - [ ] Department state transitions: `UNFUNDED → PAPER_ONLY → SHADOW_ALLOCATED`
  - [ ] Pause/resume logic with rationale
  - [ ] Promotion and demotion event logging
- [ ] CIO Agent integrated into allocation engine (rules-based v1; model-assisted v2)
- [ ] CRO Agent — firm-wide exposure aggregation and cap enforcement
- [ ] `apps/orchestrator-api/` — allocation cycle endpoint (`POST /allocation/trigger`)
- [ ] `allocation_cycles`, `allocation_decisions`, `department_status_snapshots` tables live
- [ ] `department_promotions` and governance event logging
- [ ] Allocation decision persisted as part of `DecisionTrace`
- [ ] Dashboard v0 — allocation view: department states, budgets, scorecards

### Exit Gate

```
✅ System runs a complete allocation cycle across 3 departments
✅ CapitalAllocationDecision schema-valid and persisted
✅ Department states transition correctly based on scorecard thresholds
✅ Promotion and demotion events logged to governance_events table
✅ CRO correctly rejects allocations that would breach firm-wide exposure caps
✅ Dashboard shows department states, budgets, and allocation rationale
✅ Allocation scoring is deterministic and reproducible given same inputs
```

### Service targets this phase

| Service | Output |
|---|---|
| `apps/allocation-engine/` | Full scoring + decision service |
| `apps/orchestrator-api/` | Allocation cycle endpoint |
| `apps/dashboard/` | Allocation view (v0) |
| Postgres | Allocation and governance tables live |

---

## Phase 4 — Safety Layer & Wallet Authority

**Goal:** Implement the Firm Risk Core (deterministic SAE), Wallet Policy Service, and wallet authority grant model. At the end of this phase, it is architecturally impossible for any execution plan to reach a chain adapter without passing through a firm execution decision and a valid wallet authority grant. This gate must be verified by architecture tests, not by convention.

### Deliverables

- [ ] `apps/firm-risk-core/` — global risk and deterministic policy engine
  - [ ] All checks from `SPEC.md §6.1` implemented as hard gates
  - [ ] `FirmExecutionDecision` emitted: `allowed`, `checks_passed`, `checks_failed`, `verdict`
  - [ ] Circuit breaker triggers: NAV drawdown, stale data, reconciliation drift, protocol stress
  - [ ] Kill switch: `POST /control/halt` halts all execution immediately
  - [ ] Resume gate: `POST /control/resume` requires explicit re-authorization
  - [ ] All decisions logged with `cycle_id`, `checks_passed`, `checks_failed`
- [ ] `apps/wallet-policy-service/` — wallet authority and signer routing
  - [ ] Wallet class registry (master treasury, dept hot, settlement, research, yield, recovery)
  - [ ] `WalletAuthorityGrant` lifecycle: issue, use, expire, revoke
  - [ ] Grant bound to: department, chain, protocol allowlist, action types, daily limit
  - [ ] High-risk action escalation (bridge, leverage change, new protocol, large conversion)
  - [ ] All grants persisted to `wallet_authority_grants` table
  - [ ] All grant violations logged to governance events
- [ ] `config/wallets/` — wallet class and authority definitions
- [ ] `config/protocols/` — protocol allowlist definitions
- [ ] `config/chains/` — chain configuration
- [ ] Architecture tests (`tests/contract/safety/`)
  - [ ] Proves no execution request reaches an adapter without `FirmExecutionDecision.allowed = true`
  - [ ] Proves no execution request proceeds without a valid, unexpired `WalletAuthorityGrant`
  - [ ] Proves kill switch halts all in-flight execution
  - [ ] Proves protocol allowlist blocks unlisted protocols
- [ ] `ExecutionPlan` now carries `wallet_id` and `grant_id`
- [ ] Full `DecisionTrace` populated through firm decision stage

### Exit Gate

```
✅ Architecture tests pass: no execution reaches adapter without firm decision + valid grant
✅ Kill switch (POST /control/halt) stops all execution within 1 cycle
✅ Protocol allowlist correctly blocks execution to unlisted protocols
✅ Wallet daily spend cap correctly blocks excess execution
✅ Grant expiry is enforced: expired grants cannot be used
✅ All firm decision and grant events persisted to Postgres
✅ High-risk actions (bridging, new protocol) trigger escalation correctly
```

### Service targets this phase

| Service | Output |
|---|---|
| `apps/firm-risk-core/` | Full deterministic SAE + circuit breakers |
| `apps/wallet-policy-service/` | Full wallet authority grant lifecycle |
| `config/wallets/`, `config/protocols/`, `config/chains/` | Runtime policy definitions |
| `tests/contract/safety/` | Architecture safety proof tests |
| Postgres | `wallet_authority_grants`, `wallet_balances` tables live |

---

## Phase 5 — Paper Trading & Evaluation Harness

**Goal:** Wire the full end-to-end decision cycle in paper mode: data → department pipelines → allocation → safety → execution router → paper executor → fill reconciliation → trace persistence. Implement the backtest runner, ablation harness, MLflow logging, and baseline metrics. The system should be able to run continuous paper trading cycles unattended.

### Deliverables

- [ ] `apps/execution-router/` — routes execution plans to adapters
  - [ ] Adapter selection by chain_id + protocol_id
  - [ ] Idempotency key per execution request
  - [ ] Retry policy with exponential backoff
  - [ ] Execution result fan-in
- [ ] `adapters/chains/hyperliquid/` — paper execution adapter (first rail)
  - [ ] Paper trade simulation: fills, slippage model, funding model
  - [ ] `FillReport` emitted per fill
- [ ] `apps/reconciler/` — fill reconciliation and portfolio state
  - [ ] Portfolio state updated per fill
  - [ ] NAV, unrealized PnL, realized PnL tracked per department
  - [ ] Reconciliation drift detection
  - [ ] `wallet_balances` table updated
- [ ] `apps/treasury-service/` — treasury and reserve management
  - [ ] Reserve sweep logic
  - [ ] Stablecoin conversion policy
  - [ ] Idle capital detection
  - [ ] `TreasuryEvent` logging
- [ ] `apps/optimizer-jobs/` — off-path analysis
  - [ ] Backtest runner (`jobs/backtest_runner.py`)
  - [ ] Ablation runner (`jobs/ablation_runner.py`) — single-agent vs. no-debate vs. full stack
  - [ ] Prompt policy scorer (`jobs/prompt_policy_scorer.py`)
  - [ ] MLflow experiment logging per run
- [ ] Continuous paper cycle runner — scheduled cycle trigger
- [ ] Full `DecisionTrace` populated end-to-end including fills
- [ ] Dashboard v1
  - [ ] Trace viewer (cycle → artifacts → fills)
  - [ ] Department NAV and PnL charts
  - [ ] Allocation history timeline
  - [ ] Metrics dashboard: Sharpe, drawdown, win rate per department
- [ ] `tests/simulation/` — end-to-end paper trade simulation tests

### Exit Gate

```
✅ System runs 48+ hours of continuous paper cycles unattended without errors
✅ Every cycle produces a complete, schema-valid DecisionTrace persisted to Postgres
✅ FillReports are reconciled and portfolio state stays consistent with fills
✅ Ablation metrics computed: full stack vs. single-agent vs. no-debate baseline
✅ MLflow logs all experiment runs with reproducible parameters
✅ Dashboard shows live traces, department PnL, and allocation history
✅ Reconciliation drift detection triggers correctly on injected state mismatch
```

### Service targets this phase

| Service | Output |
|---|---|
| `apps/execution-router/` | Full routing + idempotency |
| `adapters/chains/hyperliquid/` | Paper executor |
| `apps/reconciler/` | Full portfolio state |
| `apps/treasury-service/` | Reserve + yield policy |
| `apps/optimizer-jobs/` | Backtest + ablation + MLflow |
| `apps/dashboard/` | Full v1 trace + metrics dashboard |
| Postgres | All tables live; full trace schema populated |
| MLflow | Experiment tracking active |

---

## Phase 6 — Shadow Allocation & Comparison

**Goal:** Run the allocation engine in shadow mode — the executive layer shifts virtual capital across departments, generates allocation decisions, and all department execution plans are generated but not broadcast to chain. Produce a rigorous comparison of multi-department shadow allocation vs. single-strategy baseline vs. static treasury baseline over a minimum 30-day window.

### Deliverables

- [ ] Shadow mode: execution plans generated and evaluated but not broadcast
- [ ] Department state transitions into `SHADOW_ALLOCATED` tier
- [ ] Allocation engine operating on live paper metrics
- [ ] CIO agent v2 — model-assisted scoring with structured rationale
- [ ] Comparison harness (`jobs/comparison_runner.py`)
  - [ ] Multi-department shadow allocation
  - [ ] Single-strategy baseline (Momentum only, fixed capital)
  - [ ] Static treasury baseline (Treasury & Yield only)
- [ ] Metrics over 30-day window:
  - [ ] Sharpe, Sortino, Calmar
  - [ ] Max drawdown and drawdown recovery time
  - [ ] Capital utilization rate
  - [ ] Veto frequency and no-trade rate
  - [ ] Allocation cycle latency
  - [ ] Department correlation matrix
- [ ] Dashboard v2 — allocation comparison view
- [ ] Human-In-The-Loop (HITL) governance scaffold
  - [ ] `packages/governance-rules/` — HITL ruleset definitions
  - [ ] HITL gate in orchestrator: live-mode transition requires human approval
  - [ ] Human approval UI in dashboard

### Exit Gate

```
✅ Shadow allocation runs continuously for 30 days without unattended errors
✅ Comparison metrics computed and logged to MLflow for all three baselines
✅ Multi-department allocation demonstrably different from single-strategy (correlation < 0.7)
✅ Capital utilization rate documented across all three scenarios
✅ HITL gate blocks any live-mode transition without human approval in dashboard
✅ CIO agent rationale is logged and reviewable per allocation cycle
```

### Service targets this phase

| Service | Output |
|---|---|
| `apps/allocation-engine/` | Shadow mode + model-assisted CIO v2 |
| `apps/optimizer-jobs/` | Comparison runner |
| `apps/dashboard/` | Comparison + HITL approval UI (v2) |
| `packages/governance-rules/` | HITL ruleset definitions |
| MLflow | 30-day comparison experiment logged |

---

## Phase 7 — Treasury & Yield Live (Limited Capital)

**Goal:** Promote the Treasury & Yield department to live execution at a hard capital cap. This is the first real on-chain activity. Every safety gate, wallet grant, and HITL rule must be verified before and during this phase. Human approval is required for the live-mode transition.

### Deliverables

- [ ] Treasury & Yield department promoted to `LIMITED_CAPITAL` state via explicit governance event with human approval
- [ ] Live wallet infrastructure
  - [ ] Treasury hot wallet configured for yield deployment only
  - [ ] Wallet authority grants issued: limited to approved yield protocols, hard daily cap
  - [ ] Multisig or smart-account policy for amounts above threshold
- [ ] `adapters/lending/` — at least one live lending protocol adapter
- [ ] `adapters/staking/` — at least one live staking protocol adapter
- [ ] Live execution: signed transactions broadcast to chain
- [ ] Real fill reconciliation against on-chain state
- [ ] Treasury reserve sweep: idle capital above threshold moved to yield wallet
- [ ] Operational runbooks
  - [ ] `docs/runbooks/halt-and-recovery.md`
  - [ ] `docs/runbooks/incident-response.md`
  - [ ] `docs/runbooks/wallet-authority-audit.md`
  - [ ] `docs/runbooks/live-transition.md`
- [ ] `tests/chaos/` — fault injection tests
  - [ ] Stale data response
  - [ ] Adapter failure and retry
  - [ ] Reconciliation drift detection and recovery
  - [ ] Kill switch under live conditions (simulated)
- [ ] Observability stack live
  - [ ] Prometheus metrics for all services
  - [ ] Grafana dashboards: system health, execution latency, wallet balances
  - [ ] Loki log aggregation
  - [ ] Alerting: circuit breaker triggers, reconciliation drift, wallet cap approach

### Exit Gate

```
✅ Human approval recorded in governance_events for live-mode transition
✅ Treasury & Yield executing live yield deployments within hard cap
✅ All on-chain fills reconciled against Postgres portfolio state within 1 cycle
✅ Kill switch drill: halt stops all live execution within 1 cycle
✅ Stale data drill: system enters recovery mode and does not execute on stale inputs
✅ Reconciliation drift drill: drift detected and department paused automatically
✅ Runbooks reviewed and signed off
✅ Observability stack capturing all trading, process, and safety metrics
✅ 7 days of live operation without unattended incidents
```

### Service targets this phase

| Service | Output |
|---|---|
| `adapters/lending/` | Live lending adapter |
| `adapters/staking/` | Live staking adapter |
| `infra/observability/` | Prometheus + Grafana + Loki live |
| `docs/runbooks/` | All live-operation runbooks |
| `tests/chaos/` | Fault injection test suite |

---

## Phase 8 — Staged Live Expansion

**Goal:** Promote Momentum and Market Neutral departments to live execution through controlled staging: `SHADOW_ALLOCATED → LIMITED_CAPITAL`. Each promotion requires an explicit scorecard gate, governance event, and human approval.

### Deliverables

- [ ] Promotion scorecard thresholds defined for Momentum and Market Neutral
  - Minimum rolling Sharpe (e.g., ≥ 0.8 over 30 days paper + 14 days shadow)
  - Maximum drawdown (e.g., ≤ 12%)
  - Minimum regime fit score (e.g., ≥ 0.6 in current regime)
  - Minimum operational health score (e.g., ≥ 0.95)
- [ ] On-chain execution adapters
  - [ ] `adapters/chains/ethereum/` or `adapters/chains/arbitrum/` — live chain adapter
  - [ ] `adapters/dex/` — at least one live DEX adapter (Uniswap V3/V4 or GMX)
- [ ] Momentum department — `LIMITED_CAPITAL` live with conservative position caps
- [ ] Market Neutral department — `LIMITED_CAPITAL` live with delta-neutral enforcement
- [ ] Allocation engine operating with live metrics feeding into CIO scoring
- [ ] Cross-department correlation monitoring live
- [ ] Dashboard v3 — live PnL, live allocations, promotion scorecard views
- [ ] Post-trade evaluation loop
  - [ ] Slippage vs. estimate comparison per fill
  - [ ] Edge decay analysis per strategy
  - [ ] Fee burden tracking per department

### Exit Gate

```
✅ Promotion scorecard thresholds met and logged for each promoted department
✅ Governance events recorded for each promotion with human approval
✅ Both departments executing live within capital caps
✅ Cross-department correlation monitored; CRO correctly throttles correlated exposure
✅ 30 days of live operation across all three departments without unattended incidents
✅ Post-trade evaluation data confirming paper-to-live edge preservation
```

### Service targets this phase

| Service | Output |
|---|---|
| `adapters/chains/` | At least one live chain adapter |
| `adapters/dex/` | At least one live DEX adapter |
| `apps/dashboard/` | Live PnL + promotion scorecard (v3) |

---

## Phase 9 — Department Expansion

**Goal:** Onboard additional departments using the standardized department scaffold. Each new department follows the same path: incubation → paper → shadow → limited → full. The department scaffold and mandate contract are the primary artifacts.

### Deliverables

- [ ] `docs/department-guide.md` — complete guide to building and onboarding a new department
- [ ] Department scaffold generator (`Makefile` target: `make new-dept NAME=<dept-name>`)
- [ ] On-Chain Event / Flow department (`departments/onchain-event/`)
  - [ ] Governance event analyst
  - [ ] Vesting and unlock calendar data source
  - [ ] Token flow and whale behavior analyst
  - [ ] Event-driven positioning logic
- [ ] Incubation Lab (`departments/incubation/`)
  - [ ] Paper-only execution environment
  - [ ] Automated hypothesis testing
  - [ ] Research Director promotion pipeline
  - [ ] Shadow portfolio tracker
- [ ] Mean Reversion / Relative Value department (`departments/mean-reversion/`)
- [ ] Promotion pipeline for incubated strategies
  - [ ] `POST /research/promote` — Research Director promotion request
  - [ ] Promotion governance event with rationale
  - [ ] Human approval gate for live promotion

### Exit Gate

```
✅ Department scaffold generator produces a valid, bootable department in < 5 minutes
✅ On-Chain Event department running paper cycles with complete traces
✅ Incubation Lab running shadow portfolios for at least 2 experimental strategies
✅ Mean Reversion department in paper mode
✅ Promotion pipeline tested: incubated strategy successfully promoted through all tiers
```

---

## Migration from `hyperliquid-trading-firm`

The prior repo (`enuno/hyperliquid-trading-firm`) contains working implementations that should be migrated progressively, not ported wholesale. The following table maps v1 components to their v2 destination and recommended migration phase.

| v1 Component | v2 Destination | Phase | Migration Notes |
|---|---|---|---|
| `apps/orchestrator-api` | `apps/orchestrator-api` | 0–2 | Expand to firm-wide scope; preserve cycle trigger and trace patterns |
| `apps/sae-engine` | `apps/firm-risk-core` | 4 | SAE becomes broader than single-trade precheck; port deterministic check logic |
| `apps/executors` | `apps/execution-router` + `adapters/` + `apps/reconciler` | 5 | Split by responsibility: routing, adapter, reconciliation |
| `apps/treasury` | `apps/treasury-service` | 5 | Port existing treasury sweep and conversion logic |
| `apps/agents` | `departments/runtime-host/` + `departments/*/` | 2 | Agents become department-scoped via runtime host pattern |
| `packages/prompt-policies` | `packages/prompt-policies` | 1 | Preserve versioned off-path isolation |
| Research bridge / IntelliClaw | `apps/research-bridge` | 1 | Port data ingestion; preserve source isolation |
| `strategy/` | `departments/*/strategy/` | 2–3 | Align strategies with department mandates |
| HyperLiquid adapters | `adapters/chains/hyperliquid/` + `adapters/dex/` | 5–7 | HyperLiquid as one rail; preserve paper and live execution logic |
| TradingAgents submodule | `departments/runtime-host/` | 2 | Integrate framework into generic pipeline host |
| Protobuf definitions | `proto/` | 0 | Extend existing proto shapes with firm/dept/allocation/wallet types |
| MLflow + Postgres infra | Shared storage layer | 0–5 | Preserve storage patterns; add new tables per SPEC.md §10 |

---

## Metrics and Acceptance Thresholds

The following thresholds are used as gates for department promotion decisions. They are not investment targets — they are validation criteria for system behavior.

### Department Promotion Thresholds (Paper → Shadow)

| Metric | Minimum Threshold |
|---|---|
| Rolling Sharpe (30-day) | ≥ 0.6 |
| Max Drawdown | ≤ 20% |
| Win Rate | ≥ 48% |
| Trace Completeness | 100% (no missing required fields) |
| Operational Health Score | ≥ 0.90 |
| Reconciliation Drift Events | 0 unresolved |

### Department Promotion Thresholds (Shadow → Limited Capital)

| Metric | Minimum Threshold |
|---|---|
| Rolling Sharpe (30-day paper + 14-day shadow) | ≥ 0.8 |
| Max Drawdown | ≤ 15% |
| Regime Fit Score | ≥ 0.6 in current regime |
| Operational Health Score | ≥ 0.95 |
| Human Approval | Required |
| Chaos Test Results | All pass |

### System-Wide Safety Thresholds (Non-Negotiable)

| Metric | Threshold | Action |
|---|---|---|
| Firm NAV Drawdown | > 15% | Halt all departments, escalate |
| Stale Data Window | > 2 cycles | Block execution, enter recovery |
| Reconciliation Drift | Any detected | Pause affected department |
| Wallet Daily Cap Breach Attempt | Any | Block, log governance breach, alert |
| Protocol Not in Allowlist | Any | Block execution, log |

---

## Observability Requirements

All three metric classes must be live before Phase 7 (first live capital):

### Trading Metrics
`rolling_sharpe`, `rolling_sortino`, `rolling_calmar`, `max_drawdown_pct`, `win_rate`, `expectancy`, `profit_factor`, `slippage_bps_actual_vs_estimate`, `fee_burden_pct`, `capital_utilization_pct`

### Process Metrics
`cycle_latency_ms`, `debate_duration_ms`, `allocation_latency_ms`, `veto_frequency`, `no_trade_rate`, `dept_promotion_events`, `hitl_approval_latency_ms`

### Safety Metrics
`stale_data_incidents`, `policy_rejects`, `wallet_cap_violations`, `circuit_breaker_triggers`, `recovery_entries`, `manual_override_count`, `reconciliation_drift_events`, `chaos_test_pass_rate`

---

## Open Questions

The following architectural questions are not resolved and should be decided before the phase where they become blocking:

| Question | Blocking Phase | Options |
|---|---|---|
| Smart wallet vs. raw EOA for department hot wallets | Phase 4 | ERC-4337 smart account with session keys vs. HSM-backed EOA |
| Multisig provider for treasury vault | Phase 4 | Safe (Gnosis Safe), custom multisig, or hardware-backed threshold signing |
| Bridge adapter risk model | Phase 8+ | Per-bridge risk scoring, max-in-flight cap, or bridge allowlist only |
| On-chain oracle strategy | Phase 1 | Pyth, Chainlink, or DEX TWAP as primary price feed per chain |
| Cross-chain portfolio state | Phase 8+ | Single reconciler vs. per-chain reconcilers with aggregation layer |
| LLM provider redundancy | Phase 2 | Single provider with fallback model vs. multi-provider routing |
| CIO model-assisted scoring (v2) | Phase 6 | Structured reasoning prompt vs. fine-tuned allocation model |

---

> ⚠️ **Research and development status.** All strategies are hypotheses to be validated through rigorous backtesting, paper trading, and staged deployment. Nothing in this repository constitutes investment advice.

---

*Built by machines. Governed by code. Owned by no one but the keyholder.*
