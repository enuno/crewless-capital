# Crewless Capital — System Specification

**Version:** 2.1.0-draft
**Last Updated:** 2026-04-13
**Status:** Active development — research / paper trading target
**Repo:** https://github.com/enuno/crewless-capital
**Changelog:** v2.1.0 — closes all open architectural questions: wallet provider (ERC-4337 + Safe), oracle stack (Pyth/Chainlink/TWAP), bridge policy, LLM redundancy strategy, CIO agent v2 design.

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

### 5.4 CIO Agent — Model-Assisted Scoring (v2)

The CIO agent is the highest-stakes LLM in the system. It operates in two mandatory stages to prevent unconstrained model output from influencing capital allocation.

#### Stage 1 — Deterministic Pre-Filter (no LLM)

Before the CIO model is invoked, the Policy Engine removes all departments that fail hard eligibility rules. A department is ineligible if any of the following are true:

- `DepartmentStatus.state` is `PAUSED`, `SUNSETTING`, or `UNFUNDED`
- Active circuit breaker flag on the department
- Unresolved reconciliation drift event
- Stale data flag (`stale_data_incident` within last 2 cycles)
- `operational_health_score < 0.70`

The CIO model only receives the post-filter candidate set. It cannot override the pre-filter.

#### Stage 2 — Model-Assisted Scoring

The CIO agent receives a fully structured `AllocationScoringInput` JSON artifact:

```json
{
  "firm_meta": { "cycle_id": "...", "firm_mode": "SHADOW", "policy_version": "1.4.2" },
  "regime_id": "RANGING_LOW_VOL",
  "strategic_priority_id": "DEFENSIVE_YIELD",
  "current_firm_nav_usd": 500000,
  "current_allocations": [ { "department_id": "treasury-yield", "nav_usd": 50000 } ],
  "candidate_departments": [
    {
      "department_id": "treasury-yield",
      "rolling_sharpe": 1.2,
      "rolling_sortino": 1.5,
      "rolling_calmar": 2.1,
      "rolling_max_drawdown_pct": 3.1,
      "regime_fit_score": 0.91,
      "liquidity_fit_score": 0.87,
      "operational_health_score": 0.98,
      "correlation_to_firm": 0.12,
      "capital_utilization_pct": 0.72
    }
  ]
}
```

The model must return a typed `AllocationRationale` artifact containing:

- Numeric score per department per scoring dimension (§5.2)
- Proposed `DepartmentBudget` changes with explicit numeric justification
- Any departments recommended for promotion or demotion with rationale
- Confidence level in the allocation decision (`LOW` / `MEDIUM` / `HIGH`)
- Explicit acknowledgment of the regime and strategic priority constraints applied
- Any proposed changes the model cannot justify numerically must be flagged as `UNSUBSTANTIATED` and are automatically rejected by the post-check

#### Stage 3 — Deterministic Post-Check (no LLM)

After the model returns its `AllocationRationale`, the Policy Engine applies a final deterministic validation:

- Total proposed allocation must not exceed firm NAV cap
- No allocation to any pre-filtered department (model cannot override Stage 1)
- No allocation change may breach CRO exposure caps
- If CIO proposal conflicts with CRO constraint, CRO wins — unconditionally
- `UNSUBSTANTIATED` flags cause the associated budget change to be rejected; remaining valid changes proceed
- All inputs, outputs, overrides, and post-check results are included in the `DecisionTrace`

#### CIO Prompt Policy Rules

- Versioned in `packages/prompt-policies/cio/`
- Changes require a promotion gate governance event — no hot-patching
- The prompt explicitly instructs the model: it cannot modify the protocol allowlist, wallet authority, or safety layer; it cannot access data not present in the input artifact; it must cite specific metric values from the input for every proposed change

---

## 6. Risk and Safety Architecture

### 6.1 Firm Safety Layer

The Firm Safety Layer is deterministic. It is non-bypassable by any agent, operator, or department. It enforces:

- Firm NAV drawdown threshold
- Department drawdown threshold
- Gross and net exposure caps (firm and per-department)
- Per-chain and per-protocol concentration caps
- Bridge and settlement caps (see §7.4)
- Stablecoin issuer concentration
- Wallet daily spend caps
- Stale state and reconciliation drift detection
- Oracle sanity bounds (see below)
- Slippage and liquidity gates
- Protocol allowlist enforcement
- Post-incident cooldown periods
- Protocol / chain stress condition detection

#### Oracle Sanity Bounds (deterministic checks)

All price data must pass the following deterministic checks before being used in any decision. Failure triggers a `stale_data_incident` and blocks the cycle:

| Check | Condition | Action on Failure |
|---|---|---|
| Pyth confidence interval | `conf / price > 0.5%` | Block cycle; log stale_data_incident |
| Pyth–Chainlink price divergence | `abs(pyth_price - chainlink_price) / pyth_price > 1%` | Block cycle; log stale_data_incident |
| TWAP deviation from spot | TWAP diverges from Pyth spot by `> 2%` over 15-min window | Flag for CRO review; do not block unless `> 5%` |
| Heartbeat staleness | Chainlink feed not updated within its documented heartbeat × 1.5 | Demote Chainlink to tertiary; rely on Pyth only |
| Oracle source disagreement (3-way) | Pyth + Chainlink + TWAP all diverge by `> 1%` from each other | Full block; enter data recovery mode |

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
| Bridge exploit on allowlisted bridge | Immediately suspend all bridge operations; pause bridge wallet grants |

---

## 7. Wallet Authority Model

Self-custody is a constitutional invariant. Wallet architecture enforces authority boundaries programmatically via on-chain smart account policy, not by convention or database-only enforcement.

### 7.1 Wallet Classes

Each wallet class uses a specific on-chain account type. The internal `WalletPolicyService` issues grants bounded by these on-chain constraints — the on-chain policy is the enforcement layer; the service is the coordination layer.

| Class | Purpose | Account Type | Signing Model | Who Controls |
|---|---|---|---|---|
| Master Treasury Vault | Reserve capital and strategic liquidity | Safe{Wallet} (M-of-N multisig) | 2-of-3 HSM-backed signers + human operator key | Executive + policy gate; no department direct access |
| Department Hot Wallet | Production execution within budget | ERC-4337 smart account (Kernel / Biconomy Nexus) | Session key per `WalletAuthorityGrant`; revocable, scoped, time-bounded | Department, bounded by grant: chain, protocol, daily limit |
| Settlement / Bridge Wallet | Interchain transfers and settlement | ERC-4337 smart account | Session key scoped to bridge allowlist + source/destination chain pair only | Separate grant; no speculative trading permitted |
| Research / Paper Wallet | Simulation and low-risk test operations | Standard EOA (simulated) | No production signing; simulation only | No access to production treasury |
| Yield Wallet | Treasury deployment into approved yield protocols | ERC-4337 smart account | Session key scoped to yield protocol allowlist only | Treasury mandate only |
| Recovery Wallet | Emergency unwind / break-glass actions | Safe{Wallet} (2-of-3, cold) | Cold hardware keys; no hot path access | Special governance path with human approval gate |

#### Smart Account Implementation Notes

- **ERC-4337 framework:** [Kernel (ZeroDev)](https://github.com/zerodevapp/kernel) is the primary smart account implementation. [Biconomy Nexus](https://github.com/bcnmy/nexus) is the alternative/fallback.
- **Session key modules:** ERC-7579-compatible session key modules scope each `WalletAuthorityGrant` to: target contract(s), allowed method selectors, spend limit, and `validUntil` timestamp. A `WalletAuthorityGrant` in the internal database is always backed by a corresponding on-chain session key with matching constraints.
- **Safe framework:** [Safe{Core} SDK](https://github.com/safe-global/safe-core-sdk) and [Safe Transaction Service](https://github.com/safe-global/safe-transaction-service) are used for all Safe wallet operations. The Master Treasury Vault and Recovery Wallet are Safe smart accounts.
- **Chain coverage:** ERC-4337 and Safe are both deployed on Ethereum, Arbitrum, Base, and Optimism. Solana uses a native smart wallet abstraction (e.g., Squads multisig for treasury; program-derived authority for hot wallets). HyperLiquid uses its native vault/sub-account model for the HyperLiquid execution rail only.
- **Paymaster:** ERC-4337 paymasters are used for gas sponsorship on department hot wallets to simplify gas management across chains. The Wallet Policy Service manages paymaster policy.

### 7.2 Authority Stack

Every execution intent traverses this stack in order. No step can be skipped:

```
Intent
 → Department budget envelope check           [Allocation Engine]
 → Firm-wide risk engine check                [Firm Risk Core — deterministic]
 → Wallet authority grant check               [Wallet Policy Service — deterministic]
 → Protocol allowlist check                   [Policy Engine — deterministic]
 → On-chain session key validation            [Smart account — enforced on-chain]
 → Transaction builder
 → Signer / smart-account policy
 → Broadcast
 → Fill reconciliation
 → Immutable trace append
```

### 7.3 Grant Rules

- Departments never control master treasury vault keys directly.
- Wallets are bound to a specific department, chain, and protocol allowlist.
- Session keys are issued per `WalletAuthorityGrant`; grant constraints are mirrored on-chain in the session key module.
- High-risk actions (bridging, new protocol access, large treasury conversions, leverage changes) require escalated policy and may require human approval.
- Session grants expire (`validUntil`) and are revocable; all transactions carry a `grant_id` and `trace_id`.
- The Wallet Policy Service is deterministic; it does not use LLM reasoning.
- Grant issuance, use, expiry, and revocation are all persisted to the `wallet_authority_grants` table and emitted as governance events.

### 7.4 Bridge Policy

Bridge operations are the highest-risk adapter class and are subject to separate, stricter policy constraints enforced by the Policy Engine before any bridge action is permitted.

#### Bridge Allowlist

Only explicitly allowlisted bridges may be used. Additions require a human-approved governance event. The initial allowlist:

| Bridge | Rationale | Chains |
|---|---|---|
| Across Protocol | Optimistic, fast, audited, canonical for EVM↔EVM | Ethereum, Arbitrum, Base, Optimism |
| Stargate (LayerZero) | Wide chain coverage, deep liquidity | Ethereum, Arbitrum, Base, Solana |
| Wormhole NTT | Best-in-class for Solana↔EVM canonical transfers | Solana ↔ Ethereum, Arbitrum, Base |

All other bridges are blocked by default.

#### Bridge Risk Constraints (deterministic — enforced by Policy Engine)

| Constraint | Value | Notes |
|---|---|---|
| Max in-flight per bridge | 5% of firm NAV | Total value currently in transit per bridge |
| Max single bridge transaction | 2% of firm NAV | Per transaction hard cap |
| Bridge wallet daily cap | 8% of firm NAV | Aggregate across all bridges per day |
| Minimum bridge TVL | $500M | Bridge must meet TVL threshold at time of use |
| Max audit age | 18 months | Most recent full audit must be within 18 months |
| Confirmation window | Full destination confirmation required | Bridged capital is `PENDING` until destination confirms; not tradeable |
| Bridge exploit detection | Real-time monitoring via on-chain watchlist | All bridge ops suspended on any anomaly |
| Destination chain scope | Grant scoped to source + destination pair | Routing changes mid-flight not permitted |

#### Bridge Settlement Wallet Grant

A bridge action requires a `WalletAuthorityGrant` on the Settlement/Bridge Wallet class with:
- `allowed_actions: [BRIDGE_SEND]`
- `allowed_protocols: [<specific bridge id>]`
- Explicit `source_chain_id` and `destination_chain_id` in grant metadata
- `requires_human_approval: true` for amounts exceeding 1% of firm NAV

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
  RISK_VERDICT_UNSPECIFIED  = 0;
  APPROVE                   = 1;
  APPROVE_WITH_MODIFICATION = 2;
  REJECT                    = 3;
  ESCALATE                  = 4;
}

enum AllocationConfidence {
  ALLOCATION_CONFIDENCE_UNSPECIFIED = 0;
  LOW                               = 1;
  MEDIUM                            = 2;
  HIGH                              = 3;
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
  string model_tier_used     = 9;   // populated when LLM substitution occurs (see §12)
}
```

### 8.3 Department Status

```protobuf
message DepartmentStatus {
  FirmMeta        meta                     = 1;
  string          department_id            = 2;
  string          mandate_id               = 3;
  DepartmentState state                    = 4;
  double          assigned_nav_usd         = 5;
  double          utilized_nav_usd         = 6;
  double          gross_exposure_usd       = 7;
  double          net_exposure_usd         = 8;
  double          rolling_sharpe           = 9;
  double          rolling_sortino          = 10;
  double          rolling_calmar           = 11;
  double          rolling_max_drawdown_pct = 12;
  double          regime_fit_score         = 13;
  double          liquidity_fit_score      = 14;
  double          operational_health_score = 15;
  double          correlation_to_firm      = 16;
  repeated string active_risks             = 17;
  double          capital_utilization_pct  = 18;
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
  string          department_id          = 1;
  DepartmentState target_state           = 2;
  double          target_nav_usd         = 3;
  double          max_gross_exposure_usd = 4;
  double          max_net_exposure_usd   = 5;
  double          max_drawdown_pct       = 6;
  string          rationale              = 7;
}

message AllocationRationale {
  FirmMeta             meta                     = 1;
  string               regime_id                = 2;
  string               strategic_priority_id    = 3;
  AllocationConfidence confidence               = 4;
  repeated string      dimension_scores         = 5;  // serialized per-dept per-dimension scores
  repeated string      unsubstantiated_flags    = 6;  // budget changes rejected for lack of numeric justification
  string               cio_reasoning            = 7;
}

message CapitalAllocationDecision {
  FirmMeta             meta                      = 1;
  string               regime_id                 = 2;
  string               strategic_priority_id     = 3;
  repeated DepartmentBudget budgets              = 4;
  repeated string      paused_departments        = 5;
  repeated string      promoted_departments      = 6;
  repeated string      demoted_departments       = 7;
  string               allocator_rationale       = 8;
  AllocationRationale  cio_rationale             = 9;
  repeated string      cro_overrides             = 10; // budget changes rejected by CRO post-check
}
```

### 8.6 Wallet Authority Grant

```protobuf
message WalletAuthorityGrant {
  FirmMeta        meta                    = 1;
  string          grant_id                = 2;
  string          wallet_id               = 3;
  string          wallet_class            = 4;  // DEPT_HOT, SETTLEMENT, YIELD, RECOVERY, etc.
  string          department_id           = 5;
  string          chain_id                = 6;
  repeated string allowed_protocols       = 7;
  repeated string allowed_actions         = 8;
  double          max_tx_usd              = 9;
  double          max_daily_usd           = 10;
  bool            requires_multisig       = 11;
  bool            requires_human_approval = 12;
  int64           expires_at_ms           = 13;
  string          session_key_id          = 14; // on-chain session key backing this grant
  string          source_chain_id         = 15; // bridge grants only
  string          destination_chain_id    = 16; // bridge grants only
}
```

### 8.7 Execution Plan and Firm Decision

```protobuf
message OnchainAction {
  string chain_id         = 1;
  string protocol_id      = 2;
  string action_type      = 3;
  string input_asset      = 4;
  string output_asset     = 5;
  double amount_usd       = 6;
  double max_slippage_bps = 7;
  bool   reduce_only      = 8;
}

message ExecutionPlan {
  FirmMeta             meta          = 1;
  string               department_id = 2;
  string               wallet_id     = 3;
  string               grant_id      = 4;
  string               session_key_id = 5;
  repeated OnchainAction actions     = 6;
  string               plan_hash     = 7;
}

message FirmExecutionDecision {
  FirmMeta             meta              = 1;
  bool                 allowed           = 2;
  repeated string      checks_passed     = 3;
  repeated string      checks_failed     = 4;
  repeated ExecutionPlan staged_plans    = 5;
  string               rejection_reason  = 6;
  RiskVerdict          verdict           = 7;
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
  repeated string           model_substitutions = 12; // populated when failover model used
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
│       ├── across/
│       ├── stargate/
│       └── wormhole-ntt/
│
├── packages/
│   ├── schemas/                     ← Generated JSON schemas from proto
│   ├── prompt-policies/             ← Versioned prompt templates (off-path only)
│   │   └── cio/                     ← CIO agent prompt policy (versioned; promotion gate required)
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
│   ├── chains/                      ← Chain configuration + oracle assignments
│   ├── protocols/                   ← Protocol allowlist and metadata
│   ├── bridges/                     ← Bridge allowlist and risk parameters
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
│   ├── oracle-policy.md
│   ├── bridge-policy.md
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
| `cio_rationales` | AllocationRationale artifacts (CIO model output + post-check results) |
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
| `oracle_incidents` | Stale data, divergence, and sanity bound failures |
| `bridge_operations` | Bridge transaction lifecycle: initiated, in-flight, confirmed, failed |
| `model_substitution_events` | LLM provider failover events per cycle per task class |

### 10.2 Other Storage

| Store | Purpose |
|---|---|
| TimescaleDB | Time-series market data, OHLCV, funding rates, oracle prices, on-chain metrics |
| Supermemory.ai | Per-agent semantic memory via `containerTags`; RAG for research knowledge base |
| MLflow | Experiment tracking, backtest metadata, model evaluation |
| Object storage (S3-compatible) | Large artifacts: backtest outputs, archived traces |
| Redis / NATS | Transient coordination, event bus (non-durable) |

#### Memory Separation Policy

- **Hard trading state** (positions, fills, PnL, wallet balances, execution decisions) → Postgres + TimescaleDB only. Never stored in Supermemory.
- **Semantic narratives** (agent reflections, strategy reasoning summaries, regime observations, debate outcomes as narrative) → Supermemory per-agent `containerTag`.
- **Research documents and knowledge base** → Supermemory RAG (OpenClaw / research bridge integration).
- Raw market data and oracle prices → TimescaleDB only.

---

## 11. Observability and Metrics

Metrics are split into three classes:

**Trading metrics:** Sharpe, Sortino, Calmar, max drawdown, win rate, expectancy, profit factor, slippage, fee burden, exposure utilization.

**Process metrics:** Cycle latency, debate duration, allocation latency, veto frequency, no-trade frequency, department promotion rate, model substitution rate per task class.

**Safety metrics:** Stale data incidents, oracle divergence events, policy rejects, wallet authority violations, recovery entries, manual override counts, circuit breaker triggers, bridge operation status, reconciliation drift events.

---

## 12. Model Routing

LLMs are used only where probabilistic reasoning adds value. All safety-critical, authority, and policy enforcement is deterministic.

### 12.1 Provider Assignment and Failover

Each task class has a primary provider, a failover provider, and a tertiary. The `packages/model-routing/` package implements this table and enforces live-mode tier minimums.

| Task Class | Primary | Failover | Tertiary | Live-Mode Minimum Tier |
|---|---|---|---|---|
| Data normalization / entity extraction | GPT-4o-mini | Gemini Flash 2.0 | Claude Haiku 3.5 | Fast |
| Analyst synthesis | GPT-4o | Claude Sonnet 4.5 | Gemini Pro 2.0 | Mid |
| Bull/bear debate rounds | o3 | Claude Opus 4 | Gemini Ultra 2.0 | Strong reasoning |
| Trader synthesis | o3 | Claude Opus 4 | Gemini Ultra 2.0 | Strong reasoning |
| Risk committee profiles (×3 parallel) | GPT-4o | Claude Sonnet 4.5 | Gemini Pro 2.0 | Mid |
| Department portfolio agent | o3 | Claude Sonnet 4.5 | — | Mid (live); Strong (full capital) |
| CIO allocator scoring (v2) | o3 | Claude Opus 4 | — | Strong reasoning |
| SAE / Firm Safety Layer | **Deterministic rule engine** | — | — | No LLM |
| Wallet Policy Service | **Deterministic rule engine** | — | — | No LLM |

### 12.2 Failover Rules

1. **Trigger conditions for failover:** Provider returns an error, timeout (fast tier: >15s; mid tier: >60s; strong reasoning tier: >180s), or a response that fails JSON schema validation against the expected typed artifact.
2. **Schema validation gates failover:** A structurally invalid artifact (failed protobuf/JSON schema) counts as a failure and triggers failover — not only network errors.
3. **Provider health tracking:** The model router maintains a rolling 10-cycle health score per provider per task class. A provider with >20% failure rate in the rolling window is temporarily demoted to tertiary. Health score is logged to `model_substitution_events`.
4. **Budget-aware substitution:** If the primary reasoning provider is degraded, the router may downgrade to a mid-tier model and populates `FirmMeta.model_tier_used` in the artifact with the substituted tier identifier.
5. **Live-mode tier minimums are hard:** In live mode (`FirmMode.LIVE`), if no provider meets the minimum tier for a given task class, the cycle emits `HOLD_FLAT` for all affected departments and logs a `policy_reject`. Execution does not proceed on a degraded model below minimum tier.
6. **Paper and shadow mode tolerance:** In `PAPER` and `SHADOW` modes, downgraded models are acceptable and do not block cycles. Substitution events are still logged.
7. **No failover for deterministic services:** The Policy Engine and Wallet Policy Service are deterministic — they have no LLM failover path. If they are unavailable, execution halts and a circuit breaker fires.
8. **Audit trail:** Every substitution (primary → failover → tertiary) is logged in `model_substitution_events` with the task class, original provider, substituted provider, failure reason, and cycle ID. Substitutions are included in the `DecisionTrace.model_substitutions` field.

### 12.3 Per-Chain Oracle Assignment

| Chain | Primary Oracle | Secondary / Fallback | Tertiary Sanity Check |
|---|---|---|---|
| Ethereum | Pyth Network (pull) | Chainlink (push) | DEX TWAP (Uniswap V3 30-min) |
| Arbitrum | Pyth Network (pull) | Chainlink (push) | DEX TWAP (Uniswap V3 / Camelot) |
| Base | Pyth Network (pull) | Chainlink (push) | DEX TWAP (Aerodrome) |
| Solana | Pyth Network (native) | Switchboard | DEX TWAP (Orca / Raydium) |
| HyperLiquid | HyperLiquid native oracle | Pyth Network (via EVM bridge) | — |

Oracle resolution order per price request: Pyth primary → validate against Chainlink heartbeat → TWAP sanity check → apply oracle sanity bounds from §6.1 → pass to pipeline or trigger stale_data_incident.

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
8. **On-chain enforcement over database-only enforcement.** Where feasible, wallet authority constraints are enforced by on-chain session key scope — not only by the Wallet Policy Service database.

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
- CIO Allocator (rules-based v1 first, then model-assisted v2 per §5.4)
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
