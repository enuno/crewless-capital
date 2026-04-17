# MIGRATION_PLAN.md — hyperliquid-trading-firm → crewless-capital

**Version:** 1.0 — April 2026  
**Status:** Active Planning  
**Source:** `migrate-hyperliquid-firm/` (SPEC.md v3.0, AGENTS.md, DEVELOPMENT_PLAN.md, STRATEGY.md, ANALYTICS.md, RAG_IMPLEMENTATION.md, MEMORY_IMPLEMENTATION.md)  
**Target:** `crewless-capital/` multi-department autonomous trading firm  
**Authored by:** Architecture review + Crewless Capital Perplexity Space

---

## Table of Contents

1. [Migration Objective](#1-migration-objective)
2. [Source System Summary](#2-source-system-summary)
3. [Target Architecture Summary](#3-target-architecture-summary)
4. [Mapping: Source → Target](#4-mapping-source--target)
5. [What Is Adopted Directly](#5-what-is-adopted-directly)
6. [What Is Extended](#6-what-is-extended)
7. [What Is Replaced](#7-what-is-replaced)
8. [What Is Discarded](#8-what-is-discarded)
9. [New Components Required](#9-new-components-required)
10. [Repository Structure Target](#10-repository-structure-target)
11. [Protobuf Contract Migration](#11-protobuf-contract-migration)
12. [Security & Boundary Architecture](#12-security--boundary-architecture)
13. [Agent Identity & Capability Plane](#13-agent-identity--capability-plane)
14. [Phased Migration Plan](#14-phased-migration-plan)
15. [Promotion Gate Standards](#15-promotion-gate-standards)
16. [Open Questions & Decisions Required](#16-open-questions--decisions-required)

---

## 1. Migration Objective

The `hyperliquid-trading-firm` project (hereafter **HLF**) established a single-venue, single-department autonomous trading system for HyperLiquid perpetuals. It proved out the core pipeline: multi-agent LLM debate → typed protobuf contracts → deterministic SAE gate → execution → DecisionTrace audit.

**Crewless Capital (CC)** generalizes this into an institutional multi-department firm operating across multiple DEX venues and chains (Ethereum, Arbitrum, Base, Solana, HyperLiquid), with:

- A firm-wide Executive Layer (CIO, CRO, COO/CTO, Treasury, Research Director, Policy Engine)
- Multiple specialized trading departments (Momentum/Trend, Mean Reversion/RV, Market Neutral/Carry/Basis, Treasury & Yield, On-Chain Event/Flow, Incubation Lab)
- A shared Allocation Engine and Firm Risk Core (deterministic SAE, generalized)
- Multi-chain execution rails with DEX adapters (Uniswap, Curve, GMX, Jupiter, HyperLiquid)
- Self-custodied on-chain capital; zero CEX integrations
- Fully immutable DecisionTrace across all departments and venues

The migration is **additive and structural** — HLF components become the HyperLiquid department's execution rail and the reference implementation for department pipeline design. Nothing from HLF is thrown away; it is re-scoped and re-housed.

---

## 2. Source System Summary

| Dimension | HLF State |
|:--|:--|
| Venue | HyperLiquid perpetuals only |
| Agent pipeline | Observer → 5 Analysts → Bull/Bear Debate → Trader → Risk Committee (3 profiles) → FundManager → HITL Gate → SAE → Executor |
| Strategy scope | Single active strategy (paper + live dual-bot); overnight autoresearch loop |
| Risk model | Fractional Kelly sizing; SAE hard limits (drawdown, leverage, position size, liquidity, funding, event blackout) |
| Data feeds | HyperliquidFeed (REST+WS), IntelliClaw (liquidation clusters), Pyth, onchain signals |
| Quant layer | WaveDetector, QZRegimeClassifier, KellySizingService |
| Governance | OpenClaw control plane; Clawvisor HITL rulesets; operator-only live promotion |
| Audit | Immutable DecisionTrace in Postgres per cycle |
| Treasury | BTC→USDC automated conversion on realized PnL |
| Research | ResearchClaw / AutoResearchClaw 23-stage pipeline; approved hypothesis consumption |
| Memory | RAG implementation (vector store, retrieval-augmented context injection) |
| Stack | TypeScript (orchestrator, SAE), Python (agents, executors, quant), Postgres, MLflow, K8s/ArgoCD |

---

## 3. Target Architecture Summary

Crewless Capital is structured as a multi-department autonomous firm with strict constitutional invariants:

1. **Self-custody** — keys never leave operator control
2. **DEX/on-chain only** — no custodial CEX integrations
3. **Departments propose; Executive Layer allocates capital**
4. **Typed protobuf/JSON artifacts at every handoff** — no untyped prompt chains with execution authority
5. **Deterministic safety gates are non-bypassable**
6. **Strategy changes only via versioned off-path promotion gates**
7. **No-trade is a valid, measurable outcome**
8. **Every decision, allocation, and execution logged to immutable DecisionTrace**

### Firm Structure

```

┌──────────────────────────────────────────────────────────────────┐
│                        EXECUTIVE LAYER                           │
│  CIO (allocation)  │  CRO (firm risk)  │  COO/CTO (orchestration)│
│  Treasury          │  Research Director │  Policy Engine (det.)   │
└────────────────────────────┬─────────────────────────────────────┘
│ AllocationOrder
┌──────────────────┼──────────────────┐
▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌──────────────────────┐
│ Momentum/Trend  │ │ Mean Rev / RV   │ │ Market Neutral/      │
│ Dept            │ │ Dept            │ │ Carry/Basis Dept     │
└─────────────────┘ └─────────────────┘ └──────────────────────┘
▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌──────────────────────┐
│ Treasury \&      │ │ On-Chain Event/ │ │ Incubation Lab       │
│ Yield Dept      │ │ Flow Dept       │ │ (research-only tier) │
└─────────────────┘ └─────────────────┘ └──────────────────────┘
│
┌──────────────────┼──────────────────────────────┐
▼                  ▼                              ▼
┌──────────────────────────────────────────────────────────────┐
│                     SHARED SERVICES                          │
│  Allocation Engine │ Firm Risk Core (SAE) │ Wallet Policy Svc│
│  Execution Router  │ Reconciler           │ DecisionTrace     │
└──────────────────────────────────────────────────────────────┘
│
┌──────────────────┼──────────────────────────────┐
▼                  ▼                              ▼
┌──────────────────┐ ┌──────────────────┐ ┌───────────────────┐
│  Chain Adapters  │ │  DEX Adapters    │ │  Lending/Staking/ │
│  ETH/ARB/Base/   │ │  Uniswap/Curve/  │ │  LP / Bridges     │
│  Solana/HL       │ │  GMX/Jupiter/HL  │ │                   │
└──────────────────┘ └──────────────────┘ └───────────────────┘

```

### Department Internal Pipeline (per dept)

```

Observer → Analyst Team → Bull/Bear Debate → Trader/Synthesizer
→ Dept Risk → Dept Portfolio → Execution Planner
→ [Firm Risk Core gate] → [Wallet Policy gate] → Execution Router
→ Chain/DEX Adapter → DecisionTrace

```

---

## 4. Mapping: Source → Target

| HLF Component | CC Target Location | Migration Type |
|:--|:--|:--|
| `HyperliquidFeed` (REST+WS+reconcile) | `services/chain-adapters/hyperliquid/` | Adopt, generalize feed interface |
| `IntelliClaw` liquidation feed | `services/data-feeds/intelliclaw/` | Adopt as on-chain data plugin |
| `WaveDetector` + `WaveAdapter` | `services/quant/wave/` | Adopt directly |
| `QZRegimeClassifier` | `services/quant/regimes/` | Adopt, extend for multi-asset |
| `KellySizingService` | `services/quant/sizing/` | Adopt, make dept-parameterizable |
| `ObserverAgent` | Per-dept `observer/` | Clone per dept; feed sources vary by venue |
| 5 Analyst agents | Per-dept `analysts/` | Clone per dept; `onchain` analyst varies by chain |
| `BullAgent` / `BearAgent` / `NeutralAgent` | Per-dept `debate/` | Adopt pattern directly |
| `TraderAgent` | Per-dept `trader/` | Adopt; Kelly inputs from shared quant service |
| `RiskCommitteeAgent` (3 profiles) | Per-dept `dept-risk/` | Adopt as dept-level risk; firm risk is separate layer |
| `FundManager` (deterministic) | `services/allocation-engine/` (firm-level) | Promote to firm-level; dept portfolio manager is new |
| `SAE` (Safety and Execution Agent) | `services/firm-risk-core/` | Generalize; multi-venue, multi-chain policy |
| Clawvisor HITL rulesets | `services/policy-engine/` (det. + HITL) | Adopt, extend to multi-dept scope |
| `DecisionTrace` (Postgres) | `services/decision-trace/` | Adopt; add dept + chain dimensions |
| `HyperliquidExecutor` | `services/execution-router/adapters/hyperliquid/` | Adopt; new interface: `IVenueAdapter` |
| `TreasuryManager` (BTC→USDC) | `departments/treasury-yield/` | Adopt; generalize to multi-asset treasury |
| Overnight autoresearch loop | `services/research/iteration-loop/` | Adopt; scope expanded to multi-dept |
| `ResearchClaw` / `AutoResearchClaw` | `services/research/multiclaw/` | Adopt as submodule; same approval gates |
| `RAG_IMPLEMENTATION.md` | `services/research/rag/` | Adopt architecture; extend for multi-dept context |
| `MEMORY_IMPLEMENTATION.md` | `services/research/memory/` | Adopt; scope to per-dept + firm-level memory |
| `Orchestrator API` (TypeScript) | `services/orchestrator/` | Adopt; add multi-dept cycle routing |
| Paper bot / Live bot dual-bot | Per-dept promotion tiers | Adopt pattern; each dept has independent promotion |
| `AGENTS.md` scope/boundary rules | `AGENTS.md` (repo root) | Generalize; add dept-scoped write restrictions |
| `strategy_base.py` interface | `shared/strategy-base/` | Adopt; all dept strategies implement shared interface |
| `strategy_paper.py` / `strategy_live.py` | Per-dept `strategy/` | Scoped per dept; same promotion lifecycle |
| Dashboard (React/Next.js) | `apps/dashboard/` | Adopt; add multi-dept view, allocation view |
| `jobs/` (backtests, RL, OPRO) | `services/jobs/` | Adopt; add cross-dept experiment registry |
| K8s / ArgoCD / Terraform infra | `infra/` | Adopt; add per-dept namespace isolation |
| `proto/` protobuf contracts | `proto/` (repo root) | Adopt all; add firm-level and cross-dept types |
| `.claude/` + `CLAUDE.md` | `CLAUDE.md` (repo root) | Generalize for CC scope |
| `skills/` + `skills-lock.json` | `skills/` (repo root) | Adopt; extend for multi-venue research skills |

---

## 5. What Is Adopted Directly

These components transfer with minimal or no changes — only namespace and import path updates required.

### 5.1 Core Pipeline Pattern

The 8-stage decision cycle from HLF is the canonical department pipeline for CC:

```

Ingest → Quant → Observe → Analyze → Debate → Trade → Risk → Fund/Alloc
→ HITL Gate → SAE → Execute → Reconcile → Persist → Treasury → Reflect

```

Every CC department implements this pattern. The pipeline order is invariant.

### 5.2 Protobuf Data Contracts

All of `proto/common.proto`, `proto/decisioning.proto`, `proto/risk.proto`, and `proto/execution.proto` are adopted verbatim as the CC canonical contract layer. New types are added; existing types are never modified in a breaking way.

Adopted types:
- `Meta`, `Direction`, `TradeMode`, `MarketRegime` (common)
- `AnalystScore`, `ResearchPacket`, `DebateOutcome`, `TradeIntent` (decisioning)
- `RiskVote`, `RiskReview`, `ExecutionApproval`, `ExecutionApprovalRequest` (risk)
- `ExecutionRequest`, `ExecutionDecision`, `FillReport` (execution)
- `ObservationPack` (dataclass schema)

### 5.3 Safety Invariants

All 14 HLF design principles are adopted as CC design principles verbatim, with additions:

- Evidence-based and skeptical
- Separation of concerns (signal, risk, execution, audit)
- Fail-closed by default
- Immutable audit trail (DecisionTrace)
- No LLM authority over position sizing
- Deterministic safety layer (non-bypassable)
- Observable and recoverable
- Role specialization over monolithic agents
- Structured state over prompt chaining
- Adversarial challenge before commitment
- Adaptation off the hot path
- No-trade is a first-class outcome
- Live execution requires human approval (HITL)
- Treasury-aware profitability

### 5.4 Strategy Promotion Lifecycle

```

BACKTEST → (Sharpe ≥ 1.5, max_dd < 8%, n_trades ≥ 10)
→ PAPER  → (Sharpe ≥ 1.5, win_rate ≥ 45%, 48h real-time)
→ LIVE  → (real funds, drawdown guards armed)
→ RECOVERY (equity ≤ 50%: halt, Sharpe ≥ 2.0, win_rate ≥ 50%, 72h paper)

```

This lifecycle applies independently per department.

### 5.5 AGENTS.md Scope/Boundary Conventions

The HLF AGENTS.md agent boundary model — explicit MAY EDIT / MUST NOT EDIT path tables, hard rules on secrets, locked files, and research isolation — is adopted as the CC `AGENTS.md` pattern. The CC version adds dept-scoped path restrictions.

### 5.6 Closed-Bar Invariant

All quantitative analysis runs on closed bars only. This is enforced at the feed adapter level for every chain and venue. Intrabar calculations are prohibited at all layers.

### 5.7 Feed Reconciliation and Stale Data Policy

- 60-second stale threshold for all feeds
- `has_data_gap = True` propagation from feed → ObservationPack → TraderAgent → SAE
- SAE rejects any ExecutionApproval where `ObservationPack.has_data_gap = True`

### 5.8 ResearchClaw / AutoResearchClaw Integration

- Submodule structure (`multiclaw/AutoResearchClaw`, `multiclaw/researchclaw-skill`)
- `research-bridge` adapter pattern (context injector, output parser, job registry)
- `HypothesisSet.approved = true` gate — human-only, set via dashboard
- Research artifacts are READ-ONLY until approved; never written to strategy or execution paths
- Circuit breaker isolation: ResearchClaw failures must not trigger trading halts

### 5.9 RAG and Memory Architecture

The full RAG implementation (vector store, retrieval-augmented context injection for analyst and debate agents) and the memory implementation (per-agent episodic + semantic memory layers) are adopted as the shared research memory service. Scope is extended to per-dept memory namespaces plus a firm-level shared memory pool.

### 5.10 Quant Layer

`WaveDetector`, `WaveAdapter`, `QZRegimeClassifier`, and `KellySizingService` are adopted directly under `services/quant/`. All departments call the quant service over a typed RPC interface; the quant layer is shared infrastructure, not duplicated per dept.

---

## 6. What Is Extended

These components require significant generalization beyond their HLF scope.

### 6.1 SAE → Firm Risk Core

HLF's SAE is a single-venue, single-strategy deterministic gate. CC generalizes it into a **Firm Risk Core** that:

- Enforces firm-wide position limits across all departments and chains
- Enforces per-dept capital allocation caps (set by Allocation Engine)
- Enforces per-chain and per-protocol exposure limits
- Enforces firm-level drawdown circuit breakers (all departments halted on firm-level breach)
- Validates `WalletPolicyGrant` authority before any execution request
- Remains fully deterministic — no LLM in the approval path
- Still enforces all original SAE checks: leverage caps, liquidity gates, funding rate gates, stale data, event blackout, swing failure proximity

New firm-level checks added:
- Cross-dept correlation cap (prevents concentrated directional exposure across departments)
- Chain liquidity gate (per-chain, not per-asset)
- Bridge exposure limit (limits capital in transit across bridges at any time)

### 6.2 FundManager → Allocation Engine

HLF's `FundManager` is a deterministic single-strategy portfolio gate. CC promotes this to a firm-level **Allocation Engine** managed by the CIO agent:

- Receives `DeptAllocationProposal` from each active department
- Applies firm-level capital budget constraints
- Emits `AllocationOrder` per dept with `max_notional_usd` and `max_leverage`
- Enforces cross-dept correlation constraints before issuing allocations
- All allocation decisions logged to DecisionTrace with full rationale artifacts

### 6.3 Orchestrator API → Multi-Dept Orchestrator

HLF's single-cycle orchestrator becomes a multi-dept cycle router:

- Each dept runs an independent decision cycle on its own trigger cadence
- Orchestrator routes `ObservationPack` to the correct dept pipeline
- Orchestrator enforces dept pipeline ordering (no skipping stages)
- Orchestrator holds the firm-level event bus for cross-dept signals (e.g., a HYP_RISK_EVENT that all depts must receive)
- Orchestrator exposes dept-scoped endpoints in addition to firm-level endpoints

### 6.4 HyperLiquid Executor → Execution Router + Chain Adapters

HLF's HL executor is replaced by a generalized **Execution Router** with a `IVenueAdapter` interface:

```typescript
interface IVenueAdapter {
  submitOrder(req: ExecutionRequest): Promise<FillReport>;
  cancelOrder(orderId: string): Promise<void>;
  getPosition(asset: string): Promise<PositionSnapshot>;
  getBalance(): Promise<BalanceSnapshot>;
  healthCheck(): Promise<AdapterHealth>;
}
```

Adapters implemented:

- `HyperLiquidAdapter` (migrated from HLF)
- `UniswapV3Adapter` (new)
- `CurveAdapter` (new)
- `GMXAdapter` (new)
- `JupiterAdapter` (new — Solana)

The Execution Router selects the correct adapter based on `ExecutionRequest.venue` and validates against the `WalletPolicyService` before dispatch.

### 6.5 Clawvisor HITL → Policy Engine

HLF's Clawvisor HITL rulesets are generalized into a firm-level **Policy Engine** that:

- Enforces HITL approval gates per dept, per strategy tier, per capital threshold
- Manages `AgentCapabilityGrant` lifecycle (issue, rotate, revoke)
- Evaluates ClawSec-style integrity checks on `SOUL.md`, `agents.yaml`, system prompts at startup and on policy change
- Publishes signed policy bundles consumed by the Firm Risk Core
- Logs all policy changes, grant issuances, and HITL decisions to DecisionTrace


### 6.6 DecisionTrace → Multi-Dimension Audit Store

HLF's per-cycle DecisionTrace is extended with additional dimensions:

```sql
-- New columns in decision_traces table
dept_id          TEXT NOT NULL,
chain_id         TEXT NOT NULL,
venue_id         TEXT NOT NULL,
allocation_id    TEXT REFERENCES allocations(id),
wallet_grant_id  TEXT REFERENCES wallet_grants(id),
policy_version   TEXT NOT NULL,
```

Every allocation decision, policy change, wallet grant issuance, and cross-dept risk event also writes a DecisionTrace record — not just trade cycles.

### 6.7 Dashboard → Multi-Dept Operations Dashboard

Extended views:

- Per-dept P\&L, drawdown, promotion tier, strategy version
- Firm-level allocation map (capital distribution across depts and chains)
- Wallet grant status and expiry tracker
- Cross-dept correlation heatmap
- Policy Engine audit feed
- Research panel extended: per-dept hypothesis queues and approval flows


### 6.8 Treasury Manager → Treasury \& Yield Department

HLF's BTC→USDC automated treasury is promoted to a full **Treasury \& Yield Department** that:

- Manages firm-level stablecoin reserves and on-chain yield positions (lending/LP)
- Executes automated BTC→stablecoin conversion on firm-level realized PnL thresholds
- Manages bridge operations (capital rebalancing across chains)
- Reports treasury state to Allocation Engine for capital availability calculations

---

## 7. What Is Replaced

| HLF Component | Reason | CC Replacement |
| :-- | :-- | :-- |
| Single-dept single-venue pipeline | Insufficient for multi-dept multi-chain scope | Multi-dept orchestrator with per-dept isolated pipelines |
| `strategy_paper.py` / `strategy_live.py` (monorepo root) | Flat file structure doesn't scale | Per-dept `strategy/` directories with shared `BaseStrategy` interface |
| `agent/safety.py` (monolithic kill switch) | Tightly coupled to single execution path | `services/firm-risk-core/` with dept-scoped circuit breakers |
| `agent/exchange.py` (HL SDK wrapper) | Venue-specific; not extensible | `IVenueAdapter` interface + per-venue adapters |
| `agent/recovery.py` (single bot recovery) | No concept of per-dept vs. firm recovery | Per-dept recovery mode + firm-level halt (all depts) |
| `multiclaw/research-bridge` (single research pipeline) | Needs per-dept research namespacing | `services/research/` with dept-scoped job registries and hypothesis queues |


---

## 8. What Is Discarded

| HLF Component | Reason for Discard |
| :-- | :-- |
| HLF-specific single-strategy RL buffer schema | CC uses per-dept experiment registries; RL buffer schema is generalized |
| `strategy_vault.py` vault_take_pct hack | Replaced by Treasury \& Yield Department with explicit allocation rules |
| Dual-bot as top-level architectural concept | Becomes an internal implementation detail of each dept's promotion lifecycle |
| HLF-specific Makefile (empty) | Replaced by standardized per-service Makefiles and root-level dev tooling |
| HLF `trading_program.md` (high-level overview) | Superseded by CC SPEC.md |


---

## 9. New Components Required

These have no direct HLF predecessor and must be designed and built for CC.

### 9.1 SOUL.md — Firm Constitutional Document

Defines constitutional identity, mission, sovereign invariants, escalation rules, forbidden objectives, and reporting cadence for all agents. Human-only edits. ClawSec integrity check on every deploy.

### 9.2 agents.yaml — Machine-Readable Role Manifest

Per-agent machine-readable metadata:

```yaml
agents:
  - id: momentum-trader-v1
    dept: momentum-trend
    role: trader
    allowed_tools: [quant-service, debate-reader, kelly-sizer]
    allowed_repos: [dept/momentum-trend]
    allowed_environments: [paper, live]
    allowed_chains: [hyperliquid, arbitrum]
    capital_authority: none  # proposals only; allocation via Exec Layer
    escalation_to: dept-risk-momentum
    approved_by: operator
    approved_at: 2026-04-16
```


### 9.3 Wallet Policy Service

Enforces per-agent, per-chain, per-protocol, per-day capital authority grants:

```typescript
interface WalletPolicyGrant {
  grant_id: string;
  agent_id: string;
  chain_id: string;
  protocol_id: string;
  max_notional_usd_per_tx: number;
  max_daily_notional_usd: number;
  allowed_actions: ('swap' | 'lend' | 'stake' | 'lp' | 'perp')[];
  expires_at: string; // ISO8601
  issued_by: string;  // operator identity
  signed_hash: string;
}
```

No execution request proceeds without a valid, unexpired grant scoped to that agent + chain + protocol combination.

### 9.4 CIO Agent (Capital Allocation)

LLM agent in the Executive Layer that:

- Receives `DeptPerformanceReport` from each dept weekly
- Proposes `AllocationRecommendation` artifacts
- Allocation is finalized by `AllocationEngine` (deterministic) after CIO proposal
- CIO cannot directly execute capital moves; it only proposes to the Allocation Engine


### 9.5 CRO Agent (Firm Risk)

LLM advisory agent for firm-level risk that:

- Reviews cross-dept correlation reports daily
- Flags systemic risk concentrations to the Policy Engine
- Cannot override Firm Risk Core deterministic checks
- Writes `FirmRiskAdvisory` artifacts to DecisionTrace


### 9.6 Per-Dept Portfolio Manager

Deterministic (non-LLM) dept-level capital allocation within the dept's `AllocationOrder` cap. Maps to `FundManager` role in HLF pipeline but scoped per-dept and capped by firm-level allocation.

### 9.7 Cross-Dept Correlation Monitor

Background service that computes rolling cross-dept directional correlation. Emits `CorrelationAlert` when any pair exceeds the firm threshold. Consumed by Allocation Engine and Firm Risk Core.

### 9.8 Bridge Manager

Manages cross-chain capital rebalancing:

- Executes bridge operations only via human-approved `BridgeOrder` artifacts
- Tracks capital in transit; blocks further bridge operations until prior transit confirmed
- Reports bridge exposure to Firm Risk Core for real-time limit enforcement

---

## 10. Repository Structure Target

```
crewless-capital/
├── SOUL.md                          # Constitutional document — human-only edits
├── AGENTS.md                        # AI agent scope/boundary rules (repo root)
├── CLAUDE.md                        # Claude Code / coding agent guidance
├── MIGRATION_PLAN.md                # This document
├── SPEC.md                          # Crewless Capital system specification
├── agents.yaml                      # Machine-readable role manifest
│
├── proto/                           # Shared protobuf contracts
│   ├── common.proto                 # Meta, Direction, TradeMode, MarketRegime
│   ├── decisioning.proto            # AnalystScore, ResearchPacket, DebateOutcome, TradeIntent
│   ├── risk.proto                   # RiskVote, RiskReview, ExecutionApproval
│   ├── execution.proto              # ExecutionRequest, ExecutionDecision, FillReport
│   ├── allocation.proto             # AllocationOrder, DeptAllocationProposal [NEW]
│   ├── policy.proto                 # WalletPolicyGrant, AgentCapabilityGrant [NEW]
│   └── treasury.proto               # TreasuryEvent, BridgeOrder [NEW]
│
├── departments/                     # One directory per trading department
│   ├── momentum-trend/
│   │   ├── SOUL.md                  # Dept-level identity extension
│   │   ├── strategy/
│   │   │   ├── strategy_base.py     # Dept-specific BaseStrategy extension
│   │   │   ├── strategy_paper.py    # Agent-editable (≤3 param changes)
│   │   │   └── strategy_live.py     # Human-only promotion target
│   │   ├── agents/
│   │   │   ├── observer.py
│   │   │   ├── analysts/
│   │   │   ├── debate/
│   │   │   ├── trader.py
│   │   │   └── dept_portfolio.py
│   │   ├── config/
│   │   └── tests/
│   ├── mean-reversion-rv/           # Same structure
│   ├── market-neutral-carry/        # Same structure
│   ├── treasury-yield/              # Same structure + treasury extensions
│   ├── onchain-event-flow/          # Same structure + on-chain event specializations
│   └── incubation-lab/              # Research-only; no live execution path
│
├── services/                        # Shared infrastructure services
│   ├── orchestrator/                # Multi-dept cycle router (TypeScript)
│   ├── firm-risk-core/              # Generalized SAE (TypeScript, deterministic)
│   ├── allocation-engine/           # CIO + deterministic AllocationEngine
│   ├── policy-engine/               # HITL, ClawSec, grant lifecycle
│   ├── wallet-policy/               # WalletPolicyService
│   ├── execution-router/            # IVenueAdapter + per-venue adapters
│   │   └── adapters/
│   │       ├── hyperliquid/         # Migrated from HLF
│   │       ├── uniswap-v3/
│   │       ├── curve/
│   │       ├── gmx/
│   │       └── jupiter/
│   ├── chain-adapters/              # Chain-level data and tx submission
│   │   ├── hyperliquid/             # Migrated HyperliquidFeed
│   │   ├── arbitrum/
│   │   ├── base/
│   │   ├── ethereum/
│   │   └── solana/
│   ├── quant/                       # Shared quant layer (migrated from HLF)
│   │   ├── wave/                    # WaveDetector + WaveAdapter
│   │   ├── regimes/                 # QZRegimeClassifier
│   │   └── sizing/                  # KellySizingService
│   ├── decision-trace/              # Immutable audit store (Postgres)
│   ├── data-feeds/
│   │   └── intelliclaw/             # Migrated from HLF
│   ├── research/
│   │   ├── iteration-loop/          # Autoresearch loop (migrated, multi-dept)
│   │   ├── multiclaw/               # AutoResearchClaw + researchclaw-skill submodules
│   │   ├── research-bridge/         # Context injector, output parser, job registry
│   │   ├── rag/                     # RAG implementation (migrated)
│   │   └── memory/                  # Memory implementation (migrated)
│   └── jobs/                        # Backtests, RL training, OPRO prompt updates
│
├── apps/
│   └── dashboard/                   # React/Next.js multi-dept operations dashboard
│
├── shared/
│   ├── strategy-base/               # BaseStrategy interface shared by all depts
│   └── types/                       # Shared TypeScript types (generated from proto)
│
├── infra/
│   ├── k8s/                         # K8s manifests — per-dept namespace isolation
│   ├── terraform/
│   └── helm/
│
├── prompts/
│   ├── firm/                        # Firm-level agent system prompts
│   ├── departments/                 # Per-dept agent system prompts
│   └── research/                    # ResearchClaw prompt templates (migrated)
│
├── config/                          # Non-secret configuration
├── logs/
│   └── research/
│       ├── artifacts/               # ResearchClaw output artifacts (read-only)
│       └── approved_hypotheses/     # Human-approved HypothesisSet entries
│
├── migrate-hyperliquid-firm/        # Source reference (read-only; archive after migration)
│   └── [all original HLF files]
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```


---

## 11. Protobuf Contract Migration

### 11.1 Adopted Unchanged (namespace update only)

All existing HLF proto types move from `package tradingfirm.*` to `package crewless.*`. No field changes. Backward-compatible wire format.

### 11.2 New Types Required

**allocation.proto**

```protobuf
package crewless.allocation;

message DeptAllocationProposal {
  crewless.common.Meta meta           = 1;
  string               dept_id        = 2;
  double               requested_usd  = 3;
  string               rationale      = 4;
  string               strategy_version = 5;
  double               projected_sharpe = 6;
}

message AllocationOrder {
  crewless.common.Meta meta                = 1;
  string               dept_id             = 2;
  double               max_notional_usd    = 3;
  double               max_leverage        = 4;
  repeated string      allowed_chains      = 5;
  repeated string      allowed_venues      = 6;
  string               expires_at          = 7;
}
```

**policy.proto**

```protobuf
package crewless.policy;

message WalletPolicyGrant {
  string               grant_id              = 1;
  string               agent_id              = 2;
  string               chain_id              = 3;
  string               protocol_id           = 4;
  double               max_notional_per_tx   = 5;
  double               max_daily_notional    = 6;
  repeated string      allowed_actions       = 7;
  string               expires_at            = 8;
  string               issued_by             = 9;
  string               signed_hash           = 10;
}

message PolicyChangeEvent {
  crewless.common.Meta meta             = 1;
  string               policy_type      = 2;
  string               previous_version = 3;
  string               new_version      = 4;
  string               changed_by       = 5;
  string               integrity_hash   = 6;
}
```

**treasury.proto**

```protobuf
package crewless.treasury;

message TreasuryEvent {
  crewless.common.Meta meta              = 1;
  string               event_type        = 2; // conversion|bridge|yield_harvest
  string               from_asset        = 3;
  string               to_asset          = 4;
  double               amount_in         = 5;
  double               amount_out        = 6;
  string               chain_id          = 7;
  string               tx_hash           = 8;
}
```


---

## 12. Security \& Boundary Architecture

This section formalizes the three-plane security architecture for CC. See also previous architecture discussion in this Space.

### 12.1 Identity Plane

| Artifact | Controls | Enforced By |
| :-- | :-- | :-- |
| `SOUL.md` | Constitutional role, mission, forbidden objectives, escalation rules | Human ops; ClawSec integrity check on deploy |
| `agents.yaml` | Per-agent tool grants, repo access, environments, chain authority, escalation chain | Policy Engine; validated at agent startup |
| `system_prompt.md` (per agent) | Reasoning style, output schema, evidence requirements | Prompt versioning; ClawSec drift detection |
| Department SOUL extensions | Dept-specific identity constraints, strategy scope | Policy Engine; dept-scoped validation |

### 12.2 Capability Plane

| Control | Mechanism | Scope |
| :-- | :-- | :-- |
| Filesystem permissions | Per-agent Unix user; dept-scoped workspace only | Agent pods (K8s) |
| API key scoping | Per-agent, per-service, per-environment; no god tokens | K8s Secrets; rotated on schedule |
| WalletPolicyGrant | Per-agent, per-chain, per-protocol, per-day limit; signed; expiring | WalletPolicyService |
| Repo write access | Explicit MAY/MUST NOT tables in AGENTS.md; enforced by branch protection | GitHub; ArgoCD |
| Tool deny list | Per-agent allowed_tools in agents.yaml; deny-by-default | Policy Engine → Orchestrator |
| Network egress | Whitelisted endpoints per service; no open egress from execution pods | K8s NetworkPolicy |
| Secret injection | K8s Secrets only; never in code, config files, or logs | Human operator |

### 12.3 Assurance Plane

| Control | Trigger | Action on Failure |
| :-- | :-- | :-- |
| ClawSec integrity check | Deploy, startup, policy change | Block deploy; alert operator |
| Prompt/spec drift detection | Every deploy + scheduled | Alert; require human re-approval |
| Clawvisor HITL gate | Per-cycle (live mode); capital thresholds | Pause cycle; await human approval |
| DecisionTrace immutability | Every decision/allocation/execution | Write-once; any tampering = incident |
| Firm Risk Core circuit breakers | Real-time, every cycle | Halt dept or firm; no LLM override |
| Cross-dept correlation monitor | Background, continuous | CorrelationAlert → Allocation Engine |
| Bridge exposure monitor | Real-time | Block new bridge ops until cleared |
| Research circuit breaker | ResearchClaw failure | Isolate; never halt trading |

### 12.4 Hard Rules (Non-Negotiable)

These rules must be enforced in code and infrastructure, not in prompts:

1. No agent has a write path to `VAULT_SUBACCOUNT_ADDRESS`, `HL_PRIVATE_KEY`, or any wallet signing material.
2. No research artifact is written to any `strategy/`, `agent/`, or execution-path directory until `HypothesisSet.approved = true` (human-set via dashboard).
3. `strategy_live.py` (per dept) is written only by the promotion logic; never by agent code.
4. `firm-risk-core/` and `wallet-policy/` service code is human-only; never agent-editable.
5. No agent can call an execution adapter directly; all execution goes through the Execution Router → Firm Risk Core → WalletPolicyService chain.
6. Email/chat channels (AgentMail, Slack integrations) are never in the execution authority path.
7. Repo ownership does not grant deployment authority; all deploys go through ArgoCD with signed manifests.
8. Every DecisionTrace record is written before the execution it describes; never after.

---

## 13. Agent Identity \& Capability Plane

### 13.1 Role-to-Capability Matrix

| Role | Allowed Tools | Repo Write | Execution Authority | Capital Authority |
| :-- | :-- | :-- | :-- | :-- |
| Research / Analyst | quant-service, data-feeds, rag, memory | `departments/{dept}/agents/analysts/` | None | None |
| Debate (Bull/Bear/Neutral) | debate-reader, research-packet-reader | None | None | None |
| Trader | quant-service, kelly-sizer, debate-reader | `departments/{dept}/strategy/strategy_paper.py` (≤3 params) | None (proposal only) | None |
| Dept Risk | risk-models, position-reader | None | Veto (reduces size only) | None |
| Dept Portfolio | allocation-engine-reader, position-reader | None | Deterministic only | Dept cap enforcement |
| Execution Planner | execution-router (read), order-builder | None | Stages requests (not submits) | None |
| Firm Risk Core | All position data, all dept states | None (human-only code) | ALLOW / DENY (deterministic) | Enforces all caps |
| CIO Agent | Performance reports, market data | None | Proposal only | Recommends allocation |
| Allocation Engine | All dept proposals, firm risk state | None (deterministic) | Issues AllocationOrder | Firm-level budget |
| Treasury \& Yield Dept | DEX adapters (treasury scope), bridge manager | `departments/treasury-yield/strategy/` | Via Execution Router + grants | Treasury allocation only |
| Optimizer Agent | Backtest harness, RL buffer, trace reader | `departments/{dept}/strategy/strategy_paper.py` (≤3 params) | None | None |
| Research Director | All research services, hypothesis queues | `logs/research/` (read-only artifacts) | None | None |

### 13.2 Grant Lifecycle

```
Operator issues WalletPolicyGrant
  → Policy Engine validates + signs
    → Grant stored in Postgres (wallet_grants table)
      → Agent startup: Orchestrator validates grant exists and unexpired
        → Per-tx: Execution Router checks grant before dispatch
          → Daily limit: WalletPolicyService tracks rolling 24h notional
            → Expiry: Policy Engine auto-revokes; operator must re-issue
```


---

## 14. Phased Migration Plan

### Phase 0 — Foundation (Weeks 1–2)

**Objective:** Establish CC repo structure, constitutional documents, and shared contracts. No trading code moved yet.

Tasks:

- [ ] Create CC repo structure (directories per §10)
- [ ] Write `SOUL.md` (firm constitutional document)
- [ ] Write `AGENTS.md` (CC scope; dept-scoped path tables)
- [ ] Write `CLAUDE.md` (CC-scoped coding agent guidance)
- [ ] Write `agents.yaml` (initial roles: research + paper tier only)
- [ ] Migrate all HLF proto files; update namespace to `crewless.*`
- [ ] Add `allocation.proto`, `policy.proto`, `treasury.proto`
- [ ] Set up Postgres schema with multi-dept DecisionTrace dimensions
- [ ] Set up ClawSec integrity check pipeline (CI gate on SOUL.md, agents.yaml, system prompts)
- [ ] Archive `migrate-hyperliquid-firm/` as read-only reference

**Gate:** Proto contracts compile. ClawSec CI gate passes. SOUL.md, AGENTS.md, agents.yaml reviewed and signed by operator.

---

### Phase 1 — Shared Services (Weeks 3–5)

**Objective:** Migrate and generalize shared infrastructure services.

Tasks:

- [ ] Migrate `services/quant/` (WaveDetector, QZRegimeClassifier, KellySizingService) — expose typed RPC interface
- [ ] Migrate `services/data-feeds/intelliclaw/`
- [ ] Migrate `services/chain-adapters/hyperliquid/` (HyperliquidFeed)
- [ ] Migrate `services/orchestrator/` — add multi-dept cycle routing
- [ ] Build `services/execution-router/` with `IVenueAdapter` interface
- [ ] Migrate `HyperLiquidAdapter` into `execution-router/adapters/hyperliquid/`
- [ ] Build `services/wallet-policy/` (WalletPolicyService)
- [ ] Generalize `services/firm-risk-core/` from HLF SAE — add multi-chain checks
- [ ] Migrate `services/decision-trace/` — apply new schema
- [ ] Migrate `services/research/rag/` and `services/research/memory/`
- [ ] Migrate `services/research/multiclaw/` (submodules)
- [ ] Migrate `services/research/research-bridge/`
- [ ] Migrate `services/jobs/` (backtests, RL, OPRO)
- [ ] Write `shared/strategy-base/` (BaseStrategy interface)

**Gate:** All shared services have unit tests ≥ 80% coverage. Quant service passes closed-bar invariant tests. SAE passes all original HLF policy check tests. WalletPolicyService issues and validates grants correctly.

---

### Phase 2 — HyperLiquid Department (Weeks 6–8)

**Objective:** Rebuild HLF as the `momentum-trend` and/or dedicated `hyperliquid` department within CC. This is the first end-to-end dept pipeline in CC.

Tasks:

- [ ] Build `departments/momentum-trend/` full pipeline (Observer → Analysts → Debate → Trader → Dept Risk → Dept Portfolio → Execution Planner)
- [ ] Wire to shared quant service, HyperliquidFeed, IntelliClaw
- [ ] Migrate all HLF analyst agent logic into dept agents
- [ ] Migrate HLF paper bot logic into dept paper tier
- [ ] Migrate autoresearch iteration loop into `services/research/iteration-loop/`
- [ ] Build `services/allocation-engine/` (CIO agent + deterministic AllocationEngine)
- [ ] Build `services/policy-engine/` (HITL, grant lifecycle, ClawSec hooks)
- [ ] Wire full pipeline: ObservationPack → Debate → TradeIntent → RiskReview → ExecutionApproval → Firm Risk Core → WalletPolicyGrant check → Execution Router → HyperLiquidAdapter → DecisionTrace
- [ ] Migrate `apps/dashboard/` — add dept view, allocation view
- [ ] Set up K8s namespace isolation for momentum-trend dept

**Gate:** Full paper-mode cycle completes end-to-end with DecisionTrace written. Firm Risk Core rejects cycles with invalid or expired grants. HITL gate pauses cycles correctly. Promotion to paper requires human approval via dashboard.

---

### Phase 3 — Additional Chain Adapters (Weeks 9–11)

**Objective:** Expand execution rails beyond HyperLiquid.

Tasks:

- [ ] Build `chain-adapters/arbitrum/`, `chain-adapters/base/`
- [ ] Build `execution-router/adapters/uniswap-v3/`
- [ ] Build `execution-router/adapters/gmx/`
- [ ] Build `execution-router/adapters/curve/` (placeholder)
- [ ] Build `execution-router/adapters/jupiter/` (placeholder — Solana)
- [ ] Build `services/research/bridge-manager/` (cross-chain capital rebalancing)
- [ ] Extend Firm Risk Core with cross-chain exposure limits and bridge exposure monitor
- [ ] Extend WalletPolicyGrant to cover new chains and protocols

**Gate:** New adapters pass paper-mode integration tests. Firm Risk Core correctly enforces per-chain caps. Bridge Manager blocks concurrent bridge operations.

---

### Phase 4 — Additional Departments (Weeks 12–16)

**Objective:** Stand up remaining departments in paper mode.

Tasks:

- [ ] `departments/mean-reversion-rv/` — full pipeline, paper tier
- [ ] `departments/market-neutral-carry/` — full pipeline, paper tier
- [ ] `departments/treasury-yield/` — treasury + yield strategies, bridge manager integration
- [ ] `departments/onchain-event-flow/` — event-driven strategies, on-chain signal specializations
- [ ] `departments/incubation-lab/` — research-only; no execution path
- [ ] Cross-dept correlation monitor service
- [ ] CRO Agent (firm risk advisory, non-deterministic advisory layer only)
- [ ] Full multi-dept dashboard views

**Gate:** All departments run independent paper cycles. Cross-dept correlation monitor emits alerts correctly. CRO Advisory artifacts written to DecisionTrace.

---

### Phase 5 — Live Promotion \& Operations (Week 17+)

**Objective:** First live capital deployment under full CC architecture.

Tasks:

- [ ] First dept (momentum-trend/HL) promoted from paper to live after meeting promotion gate criteria
- [ ] Full HITL gate testing at live threshold
- [ ] Firm-level drawdown circuit breaker tested (paper simulation)
- [ ] Wallet grants issued by operator for live tier
- [ ] Recovery mode implemented and tested per dept
- [ ] Full audit: DecisionTrace coverage, grant lifecycle, ClawSec clean pass
- [ ] Security review: secret scanning, network policy audit, pod permission audit

**Gate:** Human operator sign-off. ClawSec passes. HITL gate verified. DecisionTrace coverage 100% for all live-path decisions.

---

## 15. Promotion Gate Standards

These are firm-wide minimums. Individual departments may set stricter thresholds.


| Gate | Metric | Threshold |
| :-- | :-- | :-- |
| BACKTEST → PAPER | Sharpe (in-sample) | ≥ 1.5 |
| BACKTEST → PAPER | Max drawdown (IS) | < 8% |
| BACKTEST → PAPER | Trade count (IS) | ≥ 10 |
| PAPER → LIVE | Sharpe (real-time paper) | ≥ 1.5 |
| PAPER → LIVE | Win rate (real-time paper) | ≥ 45% |
| PAPER → LIVE | Paper window | ≥ 48 hours |
| PAPER → LIVE | Human approval | Required (dashboard) |
| LIVE → RECOVERY | Equity vs. session start | ≤ 50% |
| RECOVERY → PAPER | Sharpe (recovery backtest) | ≥ 2.0 |
| RECOVERY → PAPER | Win rate (recovery backtest) | ≥ 50% |
| RECOVERY → PAPER | Paper window | ≥ 72 hours |
| ANY TIER | ClawSec integrity check | Must pass |
| ANY TIER | WalletPolicyGrant | Must be valid + unexpired |
| LIVE | HITL gate | Must pass per Clawvisor ruleset |


---

## 16. Open Questions \& Decisions Required

| \# | Question | Owner | Priority |
| :-- | :-- | :-- | :-- |
| 1 | Should each department have a dedicated Postgres schema or shared schema with dept_id partition key? | Architect | High |
| 2 | What is the initial firm-level capital allocation policy (equal-weight depts vs. CIO discretionary)? | Operator | High |
| 3 | Solana chain adapter: use Jupiter aggregator or direct DEX adapters (Orca, Raydium)? | Research | Medium |
| 4 | Should `incubation-lab` have a paper-only execution path, or strictly research artifacts only? | Operator | Medium |
| 5 | Cross-dept correlation cap threshold — what is the initial firm policy? | CRO / Operator | High |
| 6 | WalletPolicyGrant rotation cadence — daily expiry or event-triggered? | Security | High |
| 7 | Should the CIO Agent use strong LLM (o3/Opus) or mid-tier for allocation proposals? | Architect | Low |
| 8 | Bridge Manager: whitelist specific bridge protocols (Stargate, Across, Wormhole) or bridge-agnostic interface? | Research | Medium |
| 9 | RAG vector store: keep per-dept isolated stores or single firm-wide store with namespace filtering? | Architect | Medium |
| 10 | ClawSec integration: self-hosted or use prompt.security/clawsec hosted service? | Operator | High |
| 11 | How should the Treasury \& Yield Dept interact with on-chain lending protocols (Aave, Compound, Morpho)? | Research | Medium |
| 12 | Strategy parameter change limit (≤3 per iteration from HLF) — should this be per-dept configurable or firm-wide fixed? | Architect | Low |


---

*This document is a living plan. Update version and status on each phase gate completion. All changes to this document must be committed with a conventional commit: `docs(migration): <description>`.*

