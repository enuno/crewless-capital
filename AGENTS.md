# AGENTS.md — Crewless Capital AI Agent Development Guide

**Version:** 1.0.0
**Spec Reference:** SPEC.md v2.1.0-draft
**Repo:** https://github.com/enuno/crewless-capital

This file defines how AI coding agents (Cursor, Copilot, Claude Code, etc.) should
understand, navigate, and contribute to this codebase. Read this before modifying
any file.

---

## 1. What This Project Is

Crewless Capital is an **autonomous, self-custodied, DEX-native crypto investment
firm** modeled on institutional investment management principles, operating
exclusively on-chain. It is structured as a **multi-department AI trading firm**
where LLM agents produce typed analytical artifacts, and deterministic services
enforce all risk, wallet authority, and execution decisions.

**You are working on a financial system that controls real capital on-chain.**
Every change to safety, risk, wallet authority, or execution logic must be treated
as production-critical, regardless of current `FirmMode`.

---

## 2. Constitutional Invariants (Never Violate These)

These are non-negotiable architectural constraints. No PR, refactor, or feature
may circumvent them:

1. **No LLM output is ever directly executable trading authority.** LLMs produce
   typed reports, theses, and intents. Deterministic services (`firm-risk-core`,
   `wallet-policy-service`, `policy-engine`) enforce execution authority.
2. **All safety gates are deterministic and non-bypassable.** Do not add
   conditional bypasses, feature flags, or `if dev_mode` exceptions to the
   `firm-risk-core`, `wallet-policy-service`, or `policy-engine`.
3. **Typed artifacts at every service boundary.** Every inter-service handoff must
   use a schema-validated protobuf/JSON artifact. No raw strings, no untyped
   dicts, no prompt-chained state crossing service boundaries.
4. **Adaptation is off the hot path.** Strategy changes, prompt-policy updates, and
   optimizer outputs must go through versioned promotion gates. Never mutate live
   behavior in `FirmMode.LIVE` mid-session.
5. **Full auditability.** Every agent output, risk decision, capital allocation, and
   wallet action must be persisted to the `decision_traces` table keyed on
   `cycle_id`. Traces are append-only — never update or delete trace records.
6. **On-chain enforcement over database-only enforcement.** Wallet authority
   constraints must be enforced by on-chain session key scope (ERC-4337/ERC-7579),
   not only by the `wallet-policy-service` database.
7. **No-trade is a valid first-class outcome.** `HOLD_FLAT` must always be a
   returnable result from any department or executive agent. Never coerce an agent
   toward a trade.
8. **Environment separation is absolute.** Research, paper, shadow, and live
   environments must not share data flows without an explicit promotion gate event.

---

## 3. Repo Structure at a Glance

```

crewless-capital/
├── proto/                     # Source of truth for all typed contracts
├── apps/
│   ├── orchestrator-api/      # TypeScript — cycle coordinator, event bus
│   ├── allocation-engine/     # Python — CIO scoring and budget routing
│   ├── firm-risk-core/        # TypeScript — deterministic risk + policy engine
│   ├── wallet-policy-service/ # TypeScript — wallet authority, session keys
│   ├── execution-router/      # Python — routes plans to chain/protocol adapters
│   ├── treasury-service/      # Python — reserves, stable routing, yield
│   ├── reconciler/            # Python — portfolio state, fill reconciliation
│   ├── research-bridge/       # Python — market/on-chain data, analyst services
│   ├── optimizer-jobs/        # Python — off-path performance analysis
│   └── dashboard/             # TypeScript/Next.js — traces, approvals, governance
├── departments/
│   ├── runtime-host/          # Generic department agent pipeline host
│   ├── momentum/
│   ├── mean-reversion/
│   ├── market-neutral/
│   ├── treasury-yield/
│   ├── onchain-event/
│   └── incubation/
├── adapters/
│   ├── chains/                # Per-chain adapters (ethereum, arbitrum, base, solana, hyperliquid)
│   ├── dex/                   # DEX adapters (uniswap-v3, uniswap-v4, curve, gmx, jupiter)
│   ├── lending/
│   ├── staking/
│   └── bridges/               # across/, stargate/, wormhole-ntt/
├── packages/
│   ├── schemas/               # Generated JSON schemas from proto — do not hand-edit
│   ├── prompt-policies/       # Versioned prompt templates — changes require promotion gate
│   ├── strategy-sdk/          # Plugin API for department strategy modules
│   ├── wallet-policies/       # Wallet authority rule definitions
│   ├── governance-rules/      # HITL and promotion rulesets
│   └── model-routing/         # LLM model routing tables per task class
├── config/
│   ├── policies/              # Firm policy YAML files
│   ├── departments/           # Per-department configuration
│   ├── mandates/              # Investment mandate definitions
│   ├── chains/                # Chain config + oracle assignments
│   ├── protocols/             # Protocol allowlist and metadata
│   ├── bridges/               # Bridge allowlist and risk parameters
│   └── wallets/               # Wallet class and authority definitions
├── infra/
│   ├── k8s/
│   ├── argocd/
│   ├── terraform/
│   └── observability/         # Prometheus, Grafana, Loki configs
└── tests/
├── contract/              # Schema and typed contract tests
├── integration/
├── simulation/
└── chaos/

```

---

## 4. Language and Runtime Defaults

| Layer | Language | Notes |
|---|---|---|
| Agent pipelines, strategy, research, jobs | Python | Use `uv` for dependency management |
| Orchestrator API, risk core, wallet policy | TypeScript / Node | Strict mode; no `any` |
| Dashboard | TypeScript / Next.js | App router; server components by default |
| Chain adapters (performance-critical paths) | Python or Rust | Rust only where latency demands it |
| Infrastructure / config | Terraform, Kubernetes YAML | No manual cloud console changes |
| Proto definitions | Protobuf 3 | All contract changes start here |

### Python Conventions
- Target Python 3.11+
- Type-annotate all function signatures; use `pydantic` v2 for runtime validation
- Use `structlog` for structured JSON logging — no bare `print()` statements
- All agent outputs must be validated against their Pydantic schema before emission
- Use `pytest` with `pytest-asyncio` for async test coverage

### TypeScript Conventions
- `strict: true` in all `tsconfig.json` files — no exceptions
- Zod for runtime schema validation of incoming artifacts
- `pino` for structured logging
- No `any` — use `unknown` + type guards
- Jest for unit tests; Supertest for HTTP integration tests

---

## 5. Agent Architecture — How the Pieces Fit

### 5.1 Executive Layer Agents

These agents operate at firm-wide scope. Treat their outputs as governance events.

| Agent | App | Output Artifact | LLM? |
|---|---|---|---|
| CIO Agent | `allocation-engine` | `AllocationRationale` → `CapitalAllocationDecision` | Yes — o3 primary; hard pre/post filter |
| CRO Agent | `firm-risk-core` | `RiskVerdict` per execution plan | No — deterministic rule engine |
| COO/CTO Agent | `orchestrator-api` | Workflow routing events, health commands | Minimal — operational decisions only |
| Treasury Agent | `treasury-service` | `TreasuryEvent`, stable routing decisions | Yes — mid-tier LLM |
| Research Director | `optimizer-jobs` + `research-bridge` | Promotion recommendations | Yes — mid-tier LLM |
| Policy Engine | `firm-risk-core` | Pass/Block decisions — no rationale text | **No LLM — ever** |

### 5.2 CIO Agent — Two-Stage Model Contract (Critical)

The CIO is the highest-stakes LLM in the system. When modifying `allocation-engine`:

1. **Stage 1 (deterministic pre-filter):** The Policy Engine removes ineligible
   departments before the LLM is invoked. Ineligibility criteria:
   - `state` is `PAUSED`, `SUNSETTING`, or `UNFUNDED`
   - Active circuit breaker flag
   - Unresolved reconciliation drift event
   - `stale_data_incident` within last 2 cycles
   - `operational_health_score < 0.70`
   The model **cannot** see or override pre-filtered departments.

2. **Stage 2 (model scoring):** The model receives a fully structured
   `AllocationScoringInput` JSON artifact and must return a typed
   `AllocationRationale`. Any budget change the model cannot justify with specific
   numeric values from the input must be flagged `UNSUBSTANTIATED` in the artifact.

3. **Stage 3 (deterministic post-check):** Policy Engine validates:
   - Total allocation ≤ firm NAV cap
   - No allocation to pre-filtered departments
   - No allocation breaches CRO exposure caps (CRO wins unconditionally)
   - `UNSUBSTANTIATED` flags → reject that budget change, pass the rest

**Never merge code that allows the CIO model output to skip Stage 1 or Stage 3.**

### 5.3 Department Agent Pipeline

Every department implements the same internal pipeline (standardized comparison
is an architectural requirement, not a preference):

```

Observer / Market Context Agent
→ ResearchPacket (5 specialist analysts in parallel)
├── Technical Analyst
├── On-Chain Analyst
├── Flow / Liquidity Analyst
├── Protocol / Governance Analyst
└── News / Narrative Analyst
→ Bull / Bear Debate + Facilitator → DebateOutcome
→ Trader / Strategy Synthesizer → DepartmentIntentSet
→ Department Risk Agent → DepartmentRiskOutput
→ Department Portfolio Agent → ExecutionPlan candidates

```

When adding a new department, copy the `departments/momentum/` structure. All
departments must implement `mandate.yaml` and expose the same typed outputs.

### 5.4 Shared Services — Do Not Bypass

These services exist to centralize enforcement. Route through them, never around:

| Service | Responsibility | Bypass = ? |
|---|---|---|
| `firm-risk-core` | All risk gate enforcement | Constitution violation |
| `wallet-policy-service` | All wallet authority grants | Constitution violation |
| `reconciler` | Portfolio state and fill reconciliation | Data integrity failure |
| `research-bridge` | All market/on-chain data ingestion | Data provenance failure |
| `allocation-engine` | All department budget changes | Constitution violation |

---

## 6. Typed Contracts — The Law

**All inter-service communication is governed by `proto/` definitions.**

### Working with Protos
- All contract changes **start in `proto/`** — never hand-edit `packages/schemas/`
- After modifying a `.proto` file, regenerate schemas: `make proto-gen`
- Bump the `policy_version` in `FirmMeta` when a breaking schema change ships
- JSON shapes are isomorphic with protobuf definitions — keep them in sync

### Key Artifacts (memorize these)
| Artifact | Proto File | Description |
|---|---|---|
| `FirmMeta` | `common.proto` | Attached to every artifact; carries `cycle_id`, `trace_id`, `firm_mode` |
| `DepartmentStatus` | `department.proto` | Per-cycle department health snapshot |
| `DepartmentIntentSet` | `department.proto` | Department's proposed trades for the cycle |
| `CapitalAllocationDecision` | `allocation.proto` | CIO output: budgets, pauses, promotions |
| `WalletAuthorityGrant` | `wallet.proto` | Scoped execution authority per department/chain/protocol |
| `ExecutionPlan` | `execution.proto` | Concrete on-chain actions to execute |
| `FirmExecutionDecision` | `execution.proto` | Risk/policy verdict on an execution plan |
| `DecisionTrace` | `common.proto` | Immutable full-cycle audit record |

### Validation Rules
- Every artifact entering a service must be validated against its schema at the
  service boundary — fail loudly, never silently coerce malformed inputs
- A structurally invalid artifact (failed JSON/protobuf schema) must trigger
  failover or cycle abort — it counts as a provider failure in the model router
- `FirmMeta` must be attached to every artifact, populated before emission

---

## 7. Model Routing

LLM usage is governed by `packages/model-routing/`. When writing agent code:

| Task Class | Primary | Failover | Live Minimum |
|---|---|---|---|
| Data normalization / entity extraction | GPT-4o-mini | Gemini Flash 2.0 | Fast |
| Analyst synthesis | GPT-4o | Claude Sonnet 4.5 | Mid |
| Bull/bear debate rounds | o3 | Claude Opus 4 | Strong reasoning |
| Trader synthesis | o3 | Claude Opus 4 | Strong reasoning |
| Risk committee profiles (×3 parallel) | GPT-4o | Claude Sonnet 4.5 | Mid |
| Department portfolio agent | o3 | Claude Sonnet 4.5 | Mid (live); Strong (full capital) |
| CIO allocator scoring | o3 | Claude Opus 4 | Strong reasoning |
| Policy Engine / Wallet Policy Service | **Deterministic — no LLM** | — | — |

### Failover Rules (enforce in code)
1. Trigger failover on: provider error, timeout (fast >15s / mid >60s / strong >180s),
   or invalid schema response
2. In `FirmMode.LIVE`, if no provider meets minimum tier → emit `HOLD_FLAT` for
   all affected departments; do not proceed on a degraded model
3. In `PAPER` / `SHADOW` mode, downgraded models are acceptable but must still be
   logged to `model_substitution_events`
4. Every substitution must populate `FirmMeta.model_tier_used` and
   `DecisionTrace.model_substitutions`
5. The Policy Engine and Wallet Policy Service have **no LLM failover path** —
   if unavailable, halt execution and fire circuit breaker

---

## 8. Wallet and Execution Safety

**This section governs code touching `wallet-policy-service`, `execution-router`,
and any chain/DEX/bridge adapter.**

### Authority Stack — Every Execution Intent Must Traverse This In Order
```

Intent
→ Department budget envelope check          [allocation-engine]
→ Firm-wide risk engine check               [firm-risk-core — deterministic]
→ Wallet authority grant check              [wallet-policy-service — deterministic]
→ Protocol allowlist check                  [policy-engine — deterministic]
→ On-chain session key validation           [ERC-4337 smart account]
→ Transaction builder
→ Signer / smart-account policy
→ Broadcast
→ Fill reconciliation
→ Immutable trace append

```

No step may be skipped. No step may be reordered. Tests must assert the full
stack is exercised.

### Wallet Class Rules
- **Department Hot Wallets** are ERC-4337 smart accounts (Kernel/ZeroDev primary,
  Biconomy Nexus fallback). Session keys are scoped per `WalletAuthorityGrant`:
  target contract(s), allowed method selectors, spend limit, `validUntil`.
- **Master Treasury Vault** is a Safe{Wallet} 2-of-3 multisig. Departments never
  hold or request keys to the master vault.
- **Bridge/Settlement Wallets** require a separate `WalletAuthorityGrant` with
  `allowed_actions: [BRIDGE_SEND]` and explicit `source_chain_id` /
  `destination_chain_id`. No speculative trading permitted on this wallet class.
- All grant issuance, use, expiry, and revocation is persisted to
  `wallet_authority_grants` and emitted as a governance event.

### Bridge Policy (Hard Constraints — deterministic enforcement)
| Constraint | Value |
|---|---|
| Max in-flight per bridge | 5% of firm NAV |
| Max single bridge transaction | 2% of firm NAV |
| Bridge wallet daily cap | 8% of firm NAV |
| Minimum bridge TVL | $500M |
| Max audit age | 18 months |
| Human approval required | amounts > 1% of firm NAV |

Allowlisted bridges only: **Across Protocol**, **Stargate (LayerZero)**,
**Wormhole NTT**. All other bridges are blocked by the Policy Engine.

---

## 9. Oracle and Data Safety

All price data must pass deterministic sanity bounds before use in any decision.
Implement these checks in `research-bridge` and `firm-risk-core`:

| Check | Condition | Action |
|---|---|---|
| Pyth confidence interval | `conf / price > 0.5%` | Block cycle; log `stale_data_incident` |
| Pyth–Chainlink divergence | `abs(pyth - chainlink) / pyth > 1%` | Block cycle; log `stale_data_incident` |
| TWAP deviation from spot | TWAP vs Pyth > 2% over 15-min | Flag for CRO; block if > 5% |
| Chainlink heartbeat staleness | Feed not updated within heartbeat × 1.5 | Demote Chainlink to tertiary |
| 3-way oracle disagreement | All three diverge > 1% | Full block; enter data recovery mode |

Oracle resolution order: Pyth → validate against Chainlink heartbeat → TWAP
sanity check → apply bounds → pass to pipeline or trigger `stale_data_incident`.

A `stale_data_incident` in the last 2 cycles makes a department ineligible for
CIO scoring (see §5.2 Stage 1 pre-filter).

---

## 10. Storage Rules

### What Goes Where
| Data Type | Store | Notes |
|---|---|---|
| Positions, fills, PnL, wallet balances, execution decisions | Postgres | System of record — never in Supermemory |
| All typed artifacts, traces, governance events | Postgres | Append-only for traces |
| Time-series market data, OHLCV, funding rates, oracle prices | TimescaleDB | |
| Agent semantic memory (reflections, regime observations, debate narratives) | Supermemory.ai | Per-agent `containerTag` |
| Research knowledge base / RAG | Supermemory.ai | OpenClaw / research-bridge integration |
| Experiment tracking, backtest metadata | MLflow | |
| Large artifacts, archived traces | S3-compatible object storage | |
| Transient coordination, event bus | Redis / NATS | Non-durable; never for audit state |

### Hard Memory Separation Rule
**Hard trading state must never be stored in Supermemory.** Semantic narratives
(agent reflections, strategy reasoning summaries) must never be stored in Postgres
as the sole copy — they belong in Supermemory with a Postgres reference key.

---

## 11. Circuit Breakers and Halt Conditions

Code that triggers halts must be deterministic. When implementing triggers:

| Trigger | Required Action |
|---|---|
| Firm NAV drawdown exceeds threshold | Halt all departments; escalate to human review |
| Department drawdown exceeds budget | Pause department; reallocate capital to treasury |
| Stale data beyond threshold | Block new cycle; enter data recovery mode |
| Reconciliation drift detected | Halt affected department; trigger reconciliation job |
| Protocol exploit detected | Remove from allowlist; trigger unwind if exposed |
| Wallet policy violation attempt | Block tx; log governance breach; alert operator |
| Bridge exploit on allowlisted bridge | Suspend all bridge operations immediately; pause bridge wallet grants |
| Model router: no provider meets live minimum | Emit `HOLD_FLAT`; log `policy_reject`; do not execute |

All halt events must be persisted to `decision_traces.halt_flags` and emitted
to the observability stack (Prometheus counter + Loki log event).

---

## 12. Department Funding State Machine

Departments move between states via explicit governance events only. State
transitions must be persisted to `department_promotions`.

```

UNFUNDED → PAPER_ONLY → SHADOW_ALLOCATED → LIMITED_CAPITAL → FULL_CAPITAL
↑                ↓
THROTTLED ←───── PAUSED
↓
SUNSETTING

```

- **Research Director** can propose promotions — cannot auto-promote to live
  without a governance gate event.
- **CIO Agent** can propose budget changes — cannot set a department to
  `FULL_CAPITAL` in a single cycle if it was previously `UNFUNDED`.
- **CRO Agent** can force demotions and pauses — this overrides CIO proposals
  unconditionally.

---

## 13. Prompt Policies

All prompt templates are versioned artifacts in `packages/prompt-policies/`.

- **Never hardcode prompts in agent code.** Load them from the prompt policy store.
- **Never hot-patch a prompt policy in `FirmMode.LIVE`.** Changes require a
  promotion gate governance event logged to `governance_events`.
- The CIO prompt policy lives in `packages/prompt-policies/cio/` and is subject
  to the strictest governance — it must explicitly instruct the model that it
  cannot modify the protocol allowlist, wallet authority, or safety layer.
- Prompt policy versions are referenced in `FirmMeta.policy_version` on every
  artifact produced during a cycle.

---

## 14. Testing Requirements

### Test Coverage Expectations by Layer
| Layer | Required Test Types |
|---|---|
| `firm-risk-core` | Unit tests for every deterministic rule; chaos tests for concurrent execution |
| `wallet-policy-service` | Unit tests for every grant constraint; integration tests against local ERC-4337 fork |
| `allocation-engine` | Unit tests for pre/post filter stages; simulation tests with mock department inputs |
| Department pipelines | Contract tests asserting typed artifact outputs; simulation tests with synthetic market data |
| Chain/DEX adapters | Integration tests against local fork (Hardhat/Foundry for EVM; Bankrun for Solana) |
| Bridge adapters | Simulation tests with mock bridge responses; constraint enforcement tests |
| `reconciler` | Tests for drift detection scenarios and recovery state transitions |

### Non-Negotiable Test Rules
- Every deterministic safety gate must have a test asserting it **cannot be
  bypassed** (negative path tests are as important as happy path)
- Every typed artifact producer must have a test asserting schema validity of
  all output paths including error paths
- Circuit breaker triggers must have integration tests confirming halt propagation
- No PR touching `firm-risk-core` or `wallet-policy-service` merges without
  passing the full `tests/chaos/` suite

---

## 15. Secrets and Security

- **No hardcoded credentials, API keys, or wallet private keys — ever.**
- Secrets are injected via secret manager (AWS Secrets Manager / HashiCorp Vault)
  or controlled environment variables. The `.env.example` file defines required
  keys; `.env` files are gitignored.
- RPC endpoints with API keys must use secret-managed references, not inline URLs.
- Wallet private keys and HSM credentials are never in application code or
  environment variables in production — they are accessed via the signer service
  only.
- LLM API keys are scoped per task class in `packages/model-routing/` — use the
  routing package rather than directly instantiating LLM clients in agent code.

---

## 16. Observability

Every service must emit:
- **Structured JSON logs** via `structlog` (Python) or `pino` (TypeScript) —
  include `cycle_id`, `department_id`, `trace_id` on every log line
- **Prometheus metrics** for the three required metric classes:

  | Class | Key Metrics |
  |---|---|
  | Trading | Sharpe, Sortino, Calmar, max drawdown, win rate, slippage, fee burden, exposure utilization |
  | Process | Cycle latency, debate duration, allocation latency, veto frequency, no-trade frequency, model substitution rate |
  | Safety | Stale data incidents, oracle divergence events, policy rejects, wallet violations, circuit breaker triggers, bridge operation status, reconciliation drift events |

- Grafana dashboards live in `infra/observability/` — add panels when adding new
  metrics, do not leave metrics undashboarded.

---

## 17. Adding a New Component — Checklist

### New Department
- [ ] Copy `departments/momentum/` as the template
- [ ] Define `mandate.yaml` with regime, typical actions, and failure modes
- [ ] Implement all 5 analyst agents and the debate/trader/risk/portfolio pipeline
- [ ] All agent outputs validate against `department.proto` schemas
- [ ] Register department in `config/departments/`
- [ ] Start in `UNFUNDED` state — never deploy directly to `LIMITED_CAPITAL`
- [ ] Add simulation tests in `tests/simulation/`

### New Chain Adapter
- [ ] Create under `adapters/chains/<chain-name>/`
- [ ] Add chain config and oracle assignments to `config/chains/`
- [ ] Implement oracle sanity bounds checks (§9) for the chain's oracle set
- [ ] Register in protocol allowlist before any department can use it
- [ ] Integration tests against a local fork

### New DEX / Protocol Adapter
- [ ] Create under `adapters/dex/<protocol-name>/`
- [ ] Add to `config/protocols/` allowlist with metadata (audit date, TVL source)
- [ ] Human governance event required to add to active allowlist
- [ ] Never accessible until a `WalletAuthorityGrant` explicitly includes it

### New Bridge Adapter
- [ ] Create under `adapters/bridges/<bridge-name>/`
- [ ] Requires human-approved governance event to add to `config/bridges/` allowlist
- [ ] Enforce all bridge risk constraints (§8) as deterministic policy checks
- [ ] Simulate exploit detection path in `tests/chaos/`

### New Prompt Policy
- [ ] Add versioned template to `packages/prompt-policies/<agent-name>/`
- [ ] Never modify an existing version — create a new version file
- [ ] Update `packages/model-routing/` routing table if task class changes
- [ ] Promotion gate governance event required before use in `LIVE` mode

---

## 18. PR and Commit Standards

- Commit messages follow Conventional Commits:
  `<type>(<scope>): <description>`
  Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `security`
  Scopes map to app/package names: `firm-risk-core`, `allocation-engine`,
  `wallet-policy`, `dept-momentum`, `adapter-across`, etc.

- PRs touching constitutional invariants (§2) require explicit acknowledgment in
  the PR description: *"This change does not bypass any constitutional invariant
  because..."*

- PRs touching `firm-risk-core`, `wallet-policy-service`, or `policy-engine`
  require passing `tests/chaos/` before merge.

- PRs touching `packages/prompt-policies/` must include the governance event
  record showing promotion gate approval.

- Branch naming: `feat/<scope>/<short-description>`,
  `fix/<scope>/<short-description>`, `chore/<scope>/<short-description>`

---

## 19. MVP Scope (v2 Initial Target)

Focus development effort on these components first. Do not over-engineer
components outside MVP scope before paper trading validation is complete.

**Active departments:** Momentum, Market Neutral/Carry, Treasury & Yield

**Executive layer (MVP):** CIO Allocator (rules-based v1 first, then model-assisted
v2), CRO Risk (deterministic), Treasury Agent, Wallet Policy Service

**Infrastructure (MVP):** Orchestrator API, Allocation Engine, Firm Risk Core,
Wallet Policy Service, Execution Router, Treasury Service, Reconciler,
Dashboard (traces + allocations view)

**Validation phases before expanding scope:**
1. Research → Paper (complete traces with debate/intent outputs)
2. Paper → Shadow (allocation decisions logged and comparable)
3. Shadow → Comparison (multi-dept vs single-strategy vs static treasury, 30 days)
4. Comparison → Limited Live (Treasury & Yield, hard cap 5% NAV, human approval)
5. Limited Live → Staged Expansion (Momentum + Market Neutral via explicit scorecard)

---

> ⚠️ This system operates on real capital in production. Every agent output is a
> proposal. Only deterministic services grant execution authority. When in doubt,
> emit `HOLD_FLAT` and log a `policy_reject`.

---

*Built by machines. Governed by code. Owned by no one but the keyholder.*
