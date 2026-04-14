# Crewless Capital — System Specification

**Version:** 2.0.0-draft  
**Last Updated:** 2026-04-13  
**Status:** Active development — research / paper trading target  
**Repo:** https://github.com/enuno/crewless-capital

---

## 1. Purpose and Scope

**Crewless Capital** is an autonomous, self-custodied, DEX-native crypto investment firm modeled on the organizational and governance principles of institutional investment management (e.g. Vanguard, Fidelity), but operating exclusively on-chain across blockchain platforms, tokenized assets, and decentralized exchanges.

The system is structured as a **multi-department AI trading firm** with:

- An **Executive Team** of AI agents responsible for capital allocation, firm-wide risk, treasury governance, operations, and research promotion.
- Multiple independent **Trading Departments**, each with a defined investment mandate, internal agent team, and standardized performance reporting.
- A **Shared Services** layer providing market data, research, wallet policy, risk enforcement, portfolio state, reconciliation, and observability.
- Modular **Execution Rails** that adapt firm-level execution intent to specific chains, protocols, and DEXs.

### Constitutional Principles

The following are non-negotiable invariants enforced architecturally, not by policy suggestion:

1. **Self-custody and sovereignty first.** All production actions execute through firm-controlled wallets or smart accounts under explicit policy. No custodial exchange or third-party custody is used.
2. **DEX and on-chain only.** Crewless Capital operates exclusively across decentralized exchanges, on-chain protocols, and blockchain-native infrastructure. Custodial CEX integrations are excluded from scope by design.
3. **Department autonomy, not capital sovereignty.** Departments can propose and operate within allocated budgets, but only the Executive Layer allocates or revokes capital.
4. **Typed artifacts over free-form orchestration.** Every inter-service handoff uses structured, versioned contracts (protobuf/JSON). No untyped prompt chains cross service boundaries.
5. **Deterministic safety before execution.** Policy engines, wallet authority checks, protocol allowlists, and risk gates are non-bypassable by prompts, agents, operators, or departments.
6. **Adaptation off the hot path.** Research, strategy evolution, and optimizer outputs are proposals consumed through versioned promotion gates — they cannot mutate live behavior mid-session.
7. **No-trade and no-allocation are first-class outcomes.** The firm must be able to hold stable reserves or idle capital when no department qualifies for budget under current conditions.
8. **Full auditability.** Every agent output, risk decision, capital allocation, wallet action, and execution is persisted to an immutable decision trace.

---

## 2. Design Principles

| Principle | Implementation |
|---|---|
| Role specialization over monolithic agents | Separate executive, departmental, and shared service agents with typed interfaces |
| Structured state over prompt chaining | Typed protobuf/JSON artifacts at every handoff |
| Adversarial challenge before commitment | Bull/bear debate rounds before any TradeIntent is formed |
| Hard safety gates | Deterministic policy and wallet authority checks — non-bypassable |
| Full auditability | All artifacts keyed on `cycle_id`, persisted atomically to immutable trace |
| Adaptation off the hot path | Strategy and prompt upgrades require versioned promotion gates |
| No-trade as valid outcome | `HOLD_FLAT` emitted when edge is weak, risk unresolved, or no budget assigned |
| Human governance at the boundary | Humans define constitutions, budgets, and promotion gates; agents run within those rules |
| Zero-human company operation | All execution, monitoring, reporting, and rebalancing occurs without human discretion in the loop |

---

## 3. Organizational Structure

### 3.1 Org Chart

```
Board / Constitution Layer
└── Firm Charter + Sovereignty Principles
    └── Executive Team
        ├── CIO Agent              (capital allocation)
        ├── CRO Agent              (firm-wide risk)
        ├── COO/CTO Agent          (orchestration + operations)
        ├── Treasury Agent         (reserves, stable routing, liquidity)
        ├── Research Director      (research promotion, incubation routing)
        └── Policy Engine          (deterministic constitutional enforcement)

Trading Departments
├── Dept A: Momentum / Trend
├── Dept B: Mean Reversion / Relative Value
├── Dept C: Market Neutral / Carry / Basis
├── Dept D: Treasury & Yield
├── Dept E: On-Chain Event / Flow / Governance
└── Dept F: Incubation Lab (paper / experimental)

Shared Services
├── Market + On-Chain Data Service
├── Research Bridge
├── Portfolio / NAV / Exposure State
├── Firm Risk Core
├── Allocation Engine
├── Wallet Policy Service
├── Execution Router
├── Reconciliation Service
├── Audit / Decision Trace Store
└── Observability / Metrics / Experiments

Execution Rails
├── Chain Adapters
├── DEX Adapters
├── Lending / Staking / LP Adapters
└── Bridge / Settlement Adapters
```

### 3.2 Executive Team

| Agent | Mission | Has Authority To | Cannot Do |
|---|---|---|---|
| CIO Agent | Allocate capital across departments | Set target budgets, rebalance sleeves, pause departments | Submit execution directly |
| CRO Agent | Firm-wide risk and exposure control | Enforce aggregate caps, chain/protocol concentration, drawdown throttles | Override wallet policy or signer rules |
| COO/CTO Agent | Runtime orchestration and health | Halt/resume services, route workflows, manage degraded modes | Alter strategy logic in the hot path |
| Treasury Agent | Reserve, stable, liquidity, settlement management | Sweep reserves, manage conversion policy, deploy idle capital to yield | Open speculative positions outside mandate |
| Research Director | Promote incubated ideas through validation tiers | Move strategies from research → paper → shadow → funded | Auto-promote to live without governance gate |
| Policy Engine | Deterministic constitutional enforcement | Block any action violating firm rules | Use LLM reasoning — deterministic only |

### 3.3 Department Pattern

Each trading department implements the same internal structure, enabling standardized comparison and allocation scoring:

```
Department
├── Observer / Market Context Agent
├── Analyst Team
│   ├── Technical Analyst
│   ├── On-Chain Analyst
│   ├── Flow / Liquidity Analyst
│   ├── Protocol / Governance Analyst
│   └── News / Narrative Analyst
├── Bull / Bear Debate Agents + Facilitator
├── Trader / Strategy Synthesizer
├── Department Risk Agent
├── Department Portfolio Agent
└── Department Execution Planner
```

### 3.4 Department Taxonomy

| Department | Intended Regime | Typical Actions | Main Failure Mode |
|---|---|---|---|
| Momentum / Trend | Trend / breakout / high-participation | Directional spot, perps, momentum rotations | Whipsaw and narrative reversal |
| Mean Reversion / Relative Value | Range / overextension / liquidity snapback | Countertrend entries, dislocation fades | Catching structural breaks |
| Market Neutral / Carry / Basis | Funding / basis / carry dislocations | Delta-neutral carry, basis capture, hedged yield | Basis compression, liquidity mismatch |
| Treasury & Yield | Defensive / low-opportunity | Stable ladders, lending, staking, low-risk deployment | Smart contract or depeg risk |
| On-Chain Event / Flow | Governance, vesting, liquidity migration, whale flow | Event-driven positioning, exposure throttling | False attribution, noisy signal set |
| Incubation Lab | Research / paper / shadow | No live capital; simulated or minimally capped only | Overfitting and premature promotion |

---

## 4. Runtime Architecture

### 4.1 Layered Stack

```
[Research / Data Layer]
  → observation packs, market snapshots, protocol state, wallet state

[Department Decision Layer]
  → department research packet
  → thesis / debate artifacts
  → department trade intent set
  → department risk review
  → department execution plan proposal

[Executive Allocation Layer]
  → department scorecards
  → regime classification
  → strategic priority overlay
  → capital allocation decision (budget assignments)

[Firm Safety Layer]
  → firm-wide risk engine
  → policy engine
  → wallet authority service
  → signer policy / multisig / session-key checks

[Execution Layer]
  → execution router
  → chain / protocol adapters
  → transaction builder
  → signer service
  → broadcast / settlement / reconciliation

[Audit + State Layer]
  → portfolio state
  → decision traces
  → approvals
  → treasury events
  → governance events
  → experiments / optimizer runs
```

### 4.2 Service Map

| Service | Role | Language |
|---|---|---|
| `orchestrator-api` | Firm-wide cycle coordinator, public control plane, event bus | TypeScript / Node |
| `allocation-engine` | Executive scoring and department budget routing | Python |
| `firm-risk-core` | Global risk, policy aggregation, deterministic SAE | TypeScript / Node |
| `wallet-policy-service` | Wallet authority, signer routing, session policy | TypeScript / Node |
| `execution-router` | Route execution plans to chain/protocol adapters | Python |
| `treasury-service` | Reserve, stable, yield, settlement logic | Python |
| `reconciler` | Portfolio state, wallet balance, fill reconciliation | Python |
| `research-bridge` | Market and on-chain research ingestion, analyst services | Python |
| `optimizer-jobs` | Off-path performance analysis, promotion recommendations | Python |
| `dashboard` | Traces, approvals, allocations, governance, experiments | TypeScript / Next.js |
| `dept-runtime-host` | Generic execution host for department agent pipelines | Python |

### 4.3 Decision Cycle — End-to-End Flow

```
1. INGEST
   Market snapshot (OHLCV, order book, funding, liquidations)
   On-chain data (flows, protocol state, governance, TVL)
   Wallet and treasury state

2. DEPARTMENT ANALYZE (per active department, parallel)
   5 specialist analysts → ResearchPacket
   Bull/bear debate → DebateOutcome
   Trader synthesis → DepartmentIntentSet (trade intent proposals)
   Department risk review → DepartmentRiskOutput

3. EXECUTIVE ALLOCATE
   Department status snapshots ingested
   Regime classification
   CRO risk envelope check
   CIO allocation scoring → CapitalAllocationDecision
   Department budgets adjusted, pauses/promotions issued

4. EXECUTION PLANNING (per approved department, within budget)
   Department portfolio agent → ExecutionPlan candidates
   Firm-wide risk check (exposure caps, concentration, stress)
   Wallet authority grant check
   Firm execution decision (approved / modified / rejected)

5. EXECUTE
   Execution router fans out to chain/protocol adapters
   Transaction builder + signer service
   Broadcast and settlement
   Fill reconciliation

6. AUDIT
   DecisionTrace persisted (all artifacts, keyed on cycle_id)
   Portfolio state updated
   Metrics emitted
   Post-trade evaluation queued
```

---

## 5. Capital Allocation Model

### 5.1 Allocation Loop

Capital allocation is a firm-level portfolio process, not a trading afterthought.

1. Each department publishes a standardized `DepartmentStatus` and `DepartmentIntentSet` each cycle.
2. The Executive Layer determines regime, strategic priorities, and firm constraints.
3. The Allocation Engine scores departments across standardized dimensions.
4. A `CapitalAllocationDecision` is emitted with budget changes, pauses, and exposure envelopes.
5. Departments can only submit execution plans within their currently assigned envelope.
6. All execution plans still pass through firm-wide risk and wallet authority before chain execution.

### 5.2 Allocation Scoring Dimensions

| Dimension | Purpose |
|---|---|
| Rolling Sharpe / Sortino / Calmar | Reward risk-adjusted returns, not raw PnL |
| Drawdown slope and recovery time | Penalize brittle or slow-recovering departments |
| Regime fit score | Prevent capital in wrong-regime departments |
| Correlation to firm book | Reduce duplicated beta across departments |
| Liquidity and slippage quality | Discount alpha that disappears after fees and slippage |
| Operational health score | Penalize stale data, reconciliation drift, adapter failures |
| Strategic priority weight | Allow intentional shifts toward defense, growth, or incubation |

### 5.3 Department Funding States

Departments move between defined capital tiers. All transitions are explicit governance events:

| State | Description |
|---|---|
| `UNFUNDED` | No capital assigned; agents idle or in research mode only |
| `PAPER_ONLY` | Simulation trades only; no live capital |
| `SHADOW_ALLOCATED` | Proposals generated and evaluated but not executed |
| `LIMITED_CAPITAL` | Live with a small capped budget |
| `FULL_CAPITAL` | Full allocated budget active |
| `THROTTLED` | Budget reduced due to performance, regime, or risk trigger |
| `PAUSED` | Temporarily halted; capital returned to treasury |
| `SUNSETTING` | Winding down; positions being closed |

---

## 6. Risk and Safety Architecture

### 6.1 Firm Safety Layer

The Firm Safety Layer is deterministic. It is non-bypassable by any agent, operator, or department. It enforces:

- Firm NAV drawdown threshold
- Department drawdown threshold
- Gross and net exposure caps (firm and per-department)
- Per-chain and per-protocol concentration caps
- Bridge and settlement caps
- Stablecoin issuer concentration
- Wallet daily spend caps
- Stale state and reconciliation drift detection
- Oracle sanity bounds
- Slippage and liquidity gates
- Protocol allowlist enforcement
- Post-incident cooldown periods
- Protocol / chain stress condition detection

### 6.2 Risk Profiles

The existing aggressive / neutral / conservative risk committee pattern from prior architecture is preserved and extended. Each department operates with its own local risk profile, and the CRO agent applies an additional firm-wide constraint layer above all departments.

### 6.3 Circuit Breakers and Kill Switches

| Trigger | Action |
|---|---|
| Firm NAV drawdown exceeds threshold | Halt all departments, escalate to human review |
| Department drawdown exceeds budget | Pause department, reallocate capital |
| Stale data beyond threshold | Block new cycle, enter data recovery |
| Reconciliation drift detected | Halt affected department, trigger reconciliation job |
| Protocol exploit detected | Remove from allowlist, trigger unwind if exposed |
| Wallet policy violation attempt | Block tx, log governance breach, alert |

---

## 7. Wallet Authority Model

Self-custody is a constitutional invariant. Wallet architecture must enforce authority boundaries programmatically, not by convention.

### 7.1 Wallet Classes

| Class | Purpose | Who Controls |
|---|---|---|
| Master Treasury Vault | Reserve capital and strategic liquidity | Executive + policy gate; no department direct access |
| Department Hot Wallet | Production execution within budget | Department, bounded by grant: chain, protocol, daily limit |
| Settlement / Bridge Wallet | Interchain transfers and settlement | Separate grant; no speculative trading permitted |
| Research / Paper Wallet | Simulation and low-risk test operations | No access to production treasury |
| Yield Wallet | Treasury deployment into approved yield protocols | Treasury mandate only |
| Recovery Wallet | Emergency unwind / break-glass actions | Special governance path with human approval gate |

### 7.2 Authority Stack

Every execution intent traverses this stack in order. No step can be skipped:

```
Intent
 → Department budget envelope check
 → Firm-wide risk engine check
 → Wallet authority grant check
 → Protocol allowlist check
 → Transaction builder
 → Signer / smart-account policy
 → Broadcast
 → Fill reconciliation
 → Immutable trace append
```

### 7.3 Grant Rules

- Departments never control master treasury vault keys directly.
- Wallets are bound to a specific department, chain, and protocol allowlist.
- High-risk actions (bridging, new protocol access, large treasury conversions, leverage changes) require escalated policy and may require human approval.
- Session grants expire and are revocable; all transactions carry a `grant_id` and `trace_id`.
- The Wallet Policy Service is deterministic; it does not use LLM reasoning.

---

## 8. Typed Contracts

All inter-service communication uses protobuf-aligned typed contracts. JSON shapes are isomorphic with protobuf definitions. No untyped or free-form payloads cross service boundaries.

### 8.1 Core Enums

```protobuf
enum FirmMode {
  FIRM_MODE_UNSPECIFIED = 0;
  RESEARCH              = 1;
  PAPER                 = 2;
  SHADOW                = 3;
  LIVE                  = 4;
  RECOVERY              = 5;
}

enum DepartmentState {
  DEPT_STATE_UNSPECIFIED = 0;
  UNFUNDED               = 1;
  PAPER_ONLY             = 2;
  SHADOW_ALLOCATED       = 3;
  LIMITED_CAPITAL        = 4;
  FULL_CAPITAL           = 5;
  THROTTLED              = 6;
  PAUSED                 = 7;
  SUNSETTING             = 8;
}

enum AssetDomain {
  ASSET_DOMAIN_UNSPECIFIED = 0;
  SPOT                     = 1;
  PERP                     = 2;
  LP                       = 3;
  LENDING                  = 4;
  STAKING                  = 5;
  GOVERNANCE               = 6;
}

enum RiskVerdict {
  RISK_VERDICT_UNSPECIFIED = 0;
  APPROVE                  = 1;
  APPROVE_WITH_MODIFICATION = 2;
  REJECT                   = 3;
  ESCALATE                 = 4;
}
```

### 8.2 Firm Metadata (attached to every artifact)

```protobuf
message FirmMeta {
  string cycle_id            = 1;
  string allocation_cycle_id = 2;
  string trace_id            = 3;
  string firm_version        = 4;
  string policy_version      = 5;
  int64  created_at_ms       = 6;
  string operator_identity   = 7;
  FirmMode firm_mode         = 8;
}
```

### 8.3 Department Status

```protobuf
message DepartmentStatus {
  FirmMeta       meta                    = 1;
  string         department_id           = 2;
  string         mandate_id              = 3;
  DepartmentState state                  = 4;
  double         assigned_nav_usd        = 5;
  double         utilized_nav_usd        = 6;
  double         gross_exposure_usd      = 7;
  double         net_exposure_usd        = 8;
  double         rolling_sharpe          = 9;
  double         rolling_sortino         = 10;
  double         rolling_calmar          = 11;
  double         rolling_max_drawdown_pct = 12;
  double         regime_fit_score        = 13;
  double         liquidity_fit_score     = 14;
  double         operational_health_score = 15;
  double         correlation_to_firm     = 16;
  repeated string active_risks           = 17;
}
```

### 8.4 Department Intent Set

```protobuf
message DepartmentIntent {
  FirmMeta    meta                   = 1;
  string      department_id          = 2;
  string      strategy_id            = 3;
  string      action_type            = 4;  // BUY, SELL, LP_ADD, STAKE, BRIDGE, etc.
  string      chain_id               = 5;
  string      protocol_id            = 6;
  string      asset                  = 7;
  AssetDomain asset_domain           = 8;
  double      confidence             = 9;
  double      expected_edge_bps      = 10;
  double      requested_notional_usd = 11;
  double      max_slippage_bps       = 12;
  repeated string prerequisites      = 13;
  repeated string thesis_refs        = 14;
}

message DepartmentIntentSet {
  FirmMeta meta                     = 1;
  string   department_id            = 2;
  repeated DepartmentIntent intents = 3;
}
```

### 8.5 Capital Allocation Decision

```protobuf
message DepartmentBudget {
  string          department_id        = 1;
  DepartmentState target_state         = 2;
  double          target_nav_usd       = 3;
  double          max_gross_exposure_usd = 4;
  double          max_net_exposure_usd = 5;
  double          max_drawdown_pct     = 6;
  string          rationale            = 7;
}

message CapitalAllocationDecision {
  FirmMeta meta                         = 1;
  string   regime_id                    = 2;
  string   strategic_priority_id        = 3;
  repeated DepartmentBudget budgets     = 4;
  repeated string paused_departments    = 5;
  repeated string promoted_departments  = 6;
  repeated string demoted_departments   = 7;
  string   allocator_rationale          = 8;
}
```

### 8.6 Wallet Authority Grant

```protobuf
message WalletAuthorityGrant {
  FirmMeta meta                       = 1;
  string   grant_id                   = 2;
  string   wallet_id                  = 3;
  string   department_id              = 4;
  string   chain_id                   = 5;
  repeated string allowed_protocols   = 6;
  repeated string allowed_actions     = 7;
  double   max_tx_usd                 = 8;
  double   max_daily_usd              = 9;
  bool     requires_multisig          = 10;
  bool     requires_human_approval    = 11;
  int64    expires_at_ms              = 12;
}
```

### 8.7 Execution Plan and Firm Decision

```protobuf
message OnchainAction {
  string chain_id        = 1;
  string protocol_id     = 2;
  string action_type     = 3;
  string input_asset     = 4;
  string output_asset    = 5;
  double amount_usd      = 6;
  double max_slippage_bps = 7;
  bool   reduce_only     = 8;
}

message ExecutionPlan {
  FirmMeta meta                       = 1;
  string   department_id              = 2;
  string   wallet_id                  = 3;
  string   grant_id                   = 4;
  repeated OnchainAction actions      = 5;
  string   plan_hash                  = 6;
}

message FirmExecutionDecision {
  FirmMeta meta                          = 1;
  bool     allowed                       = 2;
  repeated string checks_passed         = 3;
  repeated string checks_failed         = 4;
  repeated ExecutionPlan staged_plans   = 5;
  string   rejection_reason             = 6;
  RiskVerdict verdict                   = 7;
}
```

### 8.8 Decision Trace

```protobuf
message DecisionTrace {
  FirmMeta                  meta              = 1;
  string                    department_id     = 2;
  repeated DepartmentStatus dept_statuses     = 3;
  CapitalAllocationDecision allocation        = 4;
  repeated DepartmentIntentSet dept_intents   = 5;
  repeated ExecutionPlan    execution_plans   = 6;
  FirmExecutionDecision     firm_decision     = 7;
  repeated FillReport       fills             = 8;
  string                    final_state       = 9;
  repeated string           halt_flags        = 10;
  int64                     latency_ms        = 11;
}
```

---

## 9. Repository Structure

```
crewless-capital/
├── README.md
├── SPEC.md                          ← This file
├── DEVELOPMENTPLAN.md
├── LICENSE
├── Makefile
├── docker-compose.yml
├── docker-compose.paper.yml
├── docker-compose.live.yml
├── .env.example
│
├── proto/
│   ├── common.proto
│   ├── market.proto
│   ├── department.proto
│   ├── allocation.proto
│   ├── risk.proto
│   ├── wallet.proto
│   ├── execution.proto
│   ├── treasury.proto
│   ├── governance.proto
│   └── research.proto
│
├── apps/
│   ├── orchestrator-api/            ← Cycle coordinator, control plane, event bus (TypeScript)
│   ├── allocation-engine/           ← Executive scoring and budget routing (Python)
│   ├── firm-risk-core/              ← Global risk, policy, deterministic SAE (TypeScript)
│   ├── wallet-policy-service/       ← Wallet authority and signer routing (TypeScript)
│   ├── execution-router/            ← Routes execution plans to chain/protocol adapters (Python)
│   ├── treasury-service/            ← Reserve, stable, yield, settlement (Python)
│   ├── reconciler/                  ← Portfolio state and fill reconciliation (Python)
│   ├── research-bridge/             ← Market/on-chain data ingestion, analyst services (Python)
│   ├── optimizer-jobs/              ← Off-path performance analysis and promotion (Python)
│   └── dashboard/                   ← Traces, approvals, allocations, governance (TypeScript/Next.js)
│
├── departments/
│   ├── runtime-host/                ← Generic department agent pipeline host
│   ├── momentum/
│   │   ├── README.md
│   │   ├── mandate.yaml
│   │   ├── analysts/
│   │   ├── debate/
│   │   ├── trader/
│   │   ├── risk/
│   │   ├── portfolio/
│   │   ├── strategy/
│   │   └── tests/
│   ├── mean-reversion/
│   ├── market-neutral/
│   ├── treasury-yield/
│   ├── onchain-event/
│   └── incubation/
│
├── adapters/
│   ├── chains/
│   │   ├── ethereum/
│   │   ├── arbitrum/
│   │   ├── base/
│   │   ├── solana/
│   │   └── hyperliquid/             ← HyperLiquid preserved as one rail among many
│   ├── dex/
│   │   ├── uniswap-v3/
│   │   ├── uniswap-v4/
│   │   ├── curve/
│   │   ├── gmx/
│   │   └── jupiter/
│   ├── lending/
│   ├── staking/
│   └── bridges/
│
├── packages/
│   ├── schemas/                     ← Generated JSON schemas from proto
│   ├── prompt-policies/             ← Versioned prompt templates (off-path only)
│   ├── strategy-sdk/                ← Plugin API for department strategy modules
│   ├── wallet-policies/             ← Wallet authority rule definitions
│   ├── governance-rules/            ← HITL and promotion rulesets
│   └── model-routing/               ← LLM model routing tables per task
│
├── config/
│   ├── env/
│   ├── policies/                    ← Firm policy YAML files
│   ├── departments/                 ← Per-department configuration
│   ├── mandates/                    ← Investment mandate definitions
│   ├── chains/                      ← Chain configuration
│   ├── protocols/                   ← Protocol allowlist and metadata
│   └── wallets/                     ← Wallet class and authority definitions
│
├── infra/
│   ├── k8s/
│   ├── argocd/
│   ├── terraform/
│   └── observability/               ← Prometheus, Grafana, Loki configs
│
├── docs/
│   ├── architecture.md
│   ├── api-contracts.md
│   ├── protobuf.md
│   ├── department-guide.md
│   ├── wallet-authority.md
│   └── runbooks/
│
└── tests/
    ├── contract/
    ├── integration/
    ├── simulation/
    └── chaos/
```

---

## 10. Storage Model

### 10.1 Postgres — System of Record

All decisions, policies, approvals, prompts, and governance state are persisted to Postgres as the canonical system of record. Redis/NATS are used only for transient coordination, never as durable audit stores.

**Core tables:**

| Table | Contents |
|---|---|
| `departments` | Department registry, mandate, state, config version |
| `department_status_snapshots` | Per-cycle DepartmentStatus snapshots |
| `allocation_cycles` | Executive allocation cycle metadata |
| `allocation_decisions` | CapitalAllocationDecision per cycle |
| `department_intents` | DepartmentIntentSet per cycle per department |
| `execution_plans` | ExecutionPlan proposals and outcomes |
| `wallet_authority_grants` | Grant lifecycle: issued, used, expired, revoked |
| `wallet_balances` | Point-in-time wallet balance snapshots |
| `protocol_exposure_snapshots` | Per-protocol exposure history |
| `chain_exposure_snapshots` | Per-chain exposure history |
| `department_nav_history` | NAV and performance history per department |
| `department_promotions` | Funding state transitions with rationale |
| `decision_traces` | Immutable cycle decision traces |
| `fills` | Execution fill records |
| `prompt_policies` | Versioned prompt templates |
| `strategy_versions` | Strategy artifact versions |
| `governance_events` | Policy changes, human approvals, overrides |
| `treasury_events` | Reserve sweeps, conversions, yield deployments |
| `hitl_rulesets` | Human-in-the-loop gate configurations |
| `human_approvals` | Human approval records |
| `recovery_state` | Incident recovery state machine |

### 10.2 Other Storage

| Store | Purpose |
|---|---|
| MLflow | Experiment tracking, backtest metadata, model evaluation |
| Object storage (S3-compatible) | Large artifacts: backtest outputs, archived traces |
| Redis / NATS | Transient coordination, event bus (non-durable) |

---

## 11. Observability and Metrics

Metrics are split into three classes:

**Trading metrics:** Sharpe, Sortino, Calmar, max drawdown, win rate, expectancy, profit factor, slippage, fee burden, exposure utilization.

**Process metrics:** Cycle latency, debate duration, allocation latency, veto frequency, no-trade frequency, department promotion rate.

**Safety metrics:** Stale data incidents, policy rejects, wallet authority violations, recovery entries, manual override counts, circuit breaker triggers.

---

## 12. Model Routing

LLMs are used only where probabilistic reasoning adds value. All safety-critical, authority, and policy enforcement is deterministic.

| Pipeline Stage | Model Tier | Rationale |
|---|---|---|
| Data normalization, entity extraction | Fast / cheap (GPT-4o-mini, Haiku) | High volume, low reasoning requirement |
| Analyst synthesis | Mid-tier (GPT-4o, Sonnet) | Domain reasoning on structured inputs |
| Bull/bear debate rounds | Strong reasoning (o3, Opus) | Adversarial argument quality matters |
| Trader synthesis | Strong reasoning | Final intent formulation |
| Risk committee profiles | Mid-tier (parallel, 3 profiles) | Profile evaluation on structured risk inputs |
| Department portfolio agent | Strong reasoning | Portfolio-level constraint enforcement |
| CIO allocator scoring | Strong reasoning | Capital allocation rationale |
| SAE / Firm Safety Layer | **Deterministic rule engine** | No LLM — hard policy only |
| Wallet Policy Service | **Deterministic rule engine** | No LLM — authority enforcement only |

---

## 13. Engineering Defaults

### 13.1 Core Rules

1. **No LLM output is ever executable trading authority on its own.** LLMs produce typed reports, theses, and intents. Deterministic services enforce risk, wallet authority, and execution.
2. **All adaptation happens off the hot path.** Strategy iteration, prompt-policy optimization, and reflection loops operate through versioned artifacts and explicit promotion gates — never silent mutation of live behavior.
3. **Typed artifacts at every handoff.** No untyped strings, no JSON blobs without schema, no prompt-chained state.
4. **Immutable trace for every cycle.** All artifacts keyed on `cycle_id`. Traces are append-only.
5. **Research, paper, and live environments are strictly separated.** No cross-environment data flow without explicit promotion gate.
6. **Least-privilege API keys and wallet grants.** Read-only keys for research; restricted, session-bounded grants for execution.
7. **Secrets management.** No hardcoded credentials. Secrets via secret manager or controlled env vars, never committed.

### 13.2 Language Defaults

| Layer | Language |
|---|---|
| Agent pipelines, strategy, research, jobs | Python |
| Orchestrator API, risk core, wallet policy | TypeScript / Node |
| Dashboard | TypeScript / Next.js |
| Chain adapters | Python (primary) or Rust (performance-critical) |
| Infrastructure / config | Terraform, Kubernetes YAML |

---

## 14. Migration from `hyperliquid-trading-firm`

The prior repo (`enuno/hyperliquid-trading-firm`) is the source of the v1 architecture and will be migrated progressively into this repo. Key mappings:

| v1 Component | v2 Destination | Notes |
|---|---|---|
| `apps/orchestrator-api` | `apps/orchestrator-api` | Keep and expand to firm-wide scope |
| `apps/sae-engine` | `apps/firm-risk-core` (split: risk core + policy engine) | SAE becomes broader than trade precheck |
| `apps/executors` | `apps/execution-router` + `adapters/*` + `apps/reconciler` | Better modularity for chain/protocol diversity |
| `apps/treasury` | `apps/treasury-service` | Preserve existing treasury work |
| `apps/agents` | `departments/*/` + `departments/runtime-host/` | Agents become department-scoped |
| `packages/prompt-policies` | `packages/prompt-policies` | Preserve off-path isolation |
| `research-bridge` | `apps/research-bridge` | Preserve existing isolation model |
| `strategy/` | `departments/*/strategy/` or `packages/strategy-sdk/` | Align with department mandates |
| HyperLiquid-specific adapters | `adapters/chains/hyperliquid/` + `adapters/dex/` | HyperLiquid is one rail among many |

---

## 15. Minimal Viable v2 Target

To avoid over-engineering before validation, the initial v2 production target is limited to:

**Three departments:**
- Momentum (paper → shadow → limited capital)
- Market Neutral / Carry (paper → shadow)
- Treasury & Yield (limited capital from day one)

**Executive layer:**
- CIO Allocator (rules-based first, then model-assisted)
- CRO Risk (deterministic)
- Treasury Agent
- Wallet Policy Service

**Infrastructure:**
- Orchestrator API
- Allocation Engine
- Firm Risk Core
- Wallet Policy Service
- Execution Router
- Treasury Service
- Reconciler
- Dashboard (traces + allocations)

This is sufficient to validate whether multi-department capital routing with standardized department reporting produces better drawdown control and capital utilization than a single-strategy baseline.

---

## 16. Validation Plan

Treat all of v2 as a hypothesis to be tested.

| Phase | Goal | Gate |
|---|---|---|
| Research | Define department contracts, build data pipeline, validate research packet quality | Analyst artifacts complete for 3 assets |
| Paper | All 3 initial departments running paper cycles with full trace persistence | Complete traces with debate and intent outputs |
| Shadow allocation | Executive allocator shifting virtual capital across departments, no live execution | Allocation decisions logged and comparable |
| Comparison | Multi-department paper vs. single-strategy paper vs. static treasury baseline | Sharpe, max drawdown, capital utilization metrics across 30 days |
| Limited live | Treasury & Yield dept live with hard cap (e.g., 5% NAV) | All safety gates validated; human approval required for live transition |
| Staged live expansion | Momentum and Market Neutral promoted through SHADOW → LIMITED_CAPITAL | Promotion governed by explicit scorecard thresholds |

---

> ⚠️ **Research and development status.** All strategies are hypotheses to be validated through rigorous backtesting, paper trading, and staged deployment. Nothing in this repository constitutes investment advice.

---

*Built by machines. Governed by code. Owned by no one but the keyholder.*
