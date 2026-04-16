# Crewless Capital Security Architecture Brief

This brief turns the earlier boundary-control discussion into a concrete, implementation-oriented security architecture for Crewless Capital. It is opinionated toward deterministic safety, typed contracts, immutable auditability, and NATS JetStream as the event backbone for inter-service orchestration.

## Objective

Design a multi-layer, multi-agent security architecture that:
- separates identity, capability, governance, and execution authority;
- prevents any LLM or prompt chain from directly exercising trading authority;
- supports department formation and promotion through controlled stages;
- uses protobuf and JSON-isomorphic artifacts for every meaningful handoff;
- uses NATS JetStream for reliable event transport, replay, and consumer isolation;
- keeps deterministic risk and wallet policy non-bypassable.

## Assumptions

- Self-custody is non-negotiable.
- DEX/on-chain only.
- Agents may propose, summarize, score, and recommend; deterministic services approve, reject, and route execution.
- Paperclip/OpenClaw/CrewAI/LangChain are orchestration and worker frameworks, not sources of execution authority.
- Clawvisor/ClawSec-style systems are security and governance augmentations, not replacements for hard policy.

---

## 1. Security model

### 1.1 Three planes

1. **Identity plane**
   - `SOUL.md`
   - `agents.yaml`
   - `system_prompt.md`
   - department charters
   - strategy manifests
   - prompt-policy versions

2. **Capability plane**
   - OpenClaw gateway policy
   - per-agent tool grants
   - repo ACLs
   - filesystem mounts
   - secret scopes
   - network egress policy
   - service-to-service authz

3. **Assurance plane**
   - CI policy validation
   - ClawSec integrity checks
   - Clawvisor runtime supervision
   - immutable DecisionTrace
   - NATS advisory and audit streams
   - reconciler and drift detection

### 1.2 Hard rules

- No LLM output is executable authority.
- No department agent may submit raw orders.
- No agent receives private signing material.
- Execution requires a deterministic `ExecutionDecision` emitted by Firm Risk Core.
- Every state transition emits a typed event and persists a DecisionTrace reference.
- No-trade is a valid terminal state.

---

## 2. Service topology

### 2.1 Core services

- **Human Creator / Executive Console**
  - approves charters, promotions, live enablement, emergency actions.

- **Hermes Supervisor**
  - supervisory agent/service for monitoring, escalation, policy drift review, anomaly triage.

- **Orchestrator API**
  - cycle coordination, state transitions, command validation, JetStream publishers.

- **Department Agent Runtime**
  - CrewAI/LangChain/OpenClaw-based staff and department workflows.

- **Policy Registry**
  - signed policies, grants, department charters, prompt-policy metadata.

- **Firm Risk Core**
  - deterministic approval engine for all execution-affecting actions.

- **Wallet Policy Service**
  - bounded authority grants by chain/protocol/department/notional window.

- **Execution Router**
  - turns approved execution plans into venue-specific actions.

- **Reconciler**
  - compares intended state vs chain-observed state and emits drift events.

- **DecisionTrace Store**
  - immutable audit record keyed by cycle, plan, approval, and tx hashes.

- **Repo Registry / GitLawb Adapter**
  - source-of-truth for agent code, manifests, prompt versions, and promotion states.

- **Security Controller**
  - Clawvisor/ClawSec integration for integrity scans, config drift alerts, and runtime posture.

### 2.2 NATS JetStream role

Use NATS JetStream as the event bus for:
- durable publication of typed artifacts;
- pull consumers for deterministic stages;
- replay during audits and incident reconstruction;
- backpressure isolation per service;
- advisory streams for dead-letter, max-delivery, and ack failures.

### 2.3 Eventing principles

- Streams store immutable facts.
- Consumers represent processing views.
- Every processing step is idempotent.
- Commands and events are separated.
- Business truth lives in Postgres/DecisionTrace, not solely in JetStream.
- JetStream is transport + replay + work coordination, not the long-term system of record.

---

## 3. JetStream subject and stream layout

### 3.1 Recommended streams

| Stream | Subjects | Purpose |
|---|---|---|
| `GOVERNANCE` | `governance.>` | approvals, promotions, policy changes, freezes |
| `AGENTS` | `agent.>` | agent lifecycle, heartbeats, outputs, failures |
| `DEPARTMENTS` | `department.>` | department proposals, charter updates, staffing events |
| `DECISIONS` | `decision.>` | research, debate, intent, risk, approval, no-trade |
| `EXECUTION` | `execution.>` | execution requests, decisions, fills, rejects |
| `SECURITY` | `security.>` | drift alerts, integrity results, policy violations |
| `RECONCILE` | `reconcile.>` | position mismatches, chain confirmations, recovery actions |
| `OBSERVABILITY` | `obs.>` | health, latency, heartbeats, metrics summaries |

### 3.2 Subject examples

- `agent.lifecycle.created`
- `agent.lifecycle.suspended`
- `agent.output.research_packet`
- `department.proposal.submitted`
- `department.charter.approved`
- `decision.trade_intent.created`
- `decision.risk_review.rejected`
- `execution.plan.approved`
- `execution.router.submitted`
- `execution.fill.reported`
- `security.integrity.failed`
- `security.policy.violation`
- `reconcile.position.drift_detected`

### 3.3 Consumer patterns

- **Pull consumers** for risk, wallet policy, execution router, reconciler.
- **Durable named consumers** for long-running services.
- **Max delivery + DLQ advisories** for poison messages.
- **Explicit ack** only after artifact persistence and idempotency checkpoint write.

---

## 4. Canonical promotion pipeline

### 4.1 Stages

1. **Research Agent**
   - may ingest data, produce reports, write code in isolated repos, run backtests.
2. **Candidate Strategy**
   - has validation artifacts, metrics, and a draft department fit.
3. **Department Agent**
   - operates inside a department with a charter and limited plan-authoring authority.
4. **Execution Planner**
   - may emit `ExecutionPlan` objects, but cannot execute them.
5. **Approved Execution Path**
   - Firm Risk Core + Wallet Policy + Execution Router only.

### 4.2 Promotion gates

| From | To | Required gates |
|---|---|---|
| Research Agent | Candidate Strategy | repo signed tag, validation report, security scan, supervisor review |
| Candidate Strategy | Department Agent | department charter approval, grants issued, prompt-policy frozen, paper-mode pass |
| Department Agent | Execution Planner | shadow run pass, no unresolved risk findings, HITL approval |
| Execution Planner | Limited Capital | deterministic risk approval, wallet grant issuance, notional caps |
| Limited Capital | Full Capital | walk-forward success, ops stability, two-person governance sign-off |

### 4.3 Stage environments

- `RESEARCH`
- `PAPER_ONLY`
- `SHADOW_ALLOCATED`
- `LIMITED_CAPITAL`
- `FULL_CAPITAL`
- `RECOVERY_ONLY`

Agents and departments may only move upward through explicit governance events.

---

## 5. Protobuf contract set

Below are compact, repo-ready starter contracts.

### 5.1 `common.proto`

```proto
syntax = "proto3";
package crewless.common;

message Meta {
  string event_id = 1;
  string correlation_id = 2;
  string trace_id = 3;
  string cycle_id = 4;
  string department_id = 5;
  string agent_id = 6;
  string actor_id = 7;
  string policy_version = 8;
  string schema_version = 9;
  int64 created_at_ms = 10;
}

enum Environment {
  ENV_UNSPECIFIED = 0;
  RESEARCH = 1;
  PAPER_ONLY = 2;
  SHADOW_ALLOCATED = 3;
  LIMITED_CAPITAL = 4;
  FULL_CAPITAL = 5;
  RECOVERY_ONLY = 6;
}

enum GrantEffect {
  GRANT_EFFECT_UNSPECIFIED = 0;
  ALLOW = 1;
  DENY = 2;
}

enum ApprovalState {
  APPROVAL_STATE_UNSPECIFIED = 0;
  PENDING = 1;
  APPROVED = 2;
  REJECTED = 3;
  EXPIRED = 4;
  REVOKED = 5;
}
```

### 5.2 `identity.proto`

```proto
syntax = "proto3";
package crewless.identity;
import "common.proto";

message AgentIdentity {
  crewless.common.Meta meta = 1;
  string agent_id = 2;
  string name = 3;
  string layer = 4;
  string role = 5;
  string owner_human_id = 6;
  string department_id = 7;
  repeated string peer_ids = 8;
  repeated string repo_ids = 9;
  repeated string tags = 10;
  crewless.common.Environment environment = 11;
  bool active = 12;
}

message DepartmentCharter {
  crewless.common.Meta meta = 1;
  string department_id = 2;
  string name = 3;
  string mission = 4;
  repeated string allowed_strategy_classes = 5;
  repeated string allowed_chains = 6;
  repeated string allowed_protocols = 7;
  repeated string allowed_instruments = 8;
  double max_gross_exposure_pct = 9;
  double max_daily_turnover_pct = 10;
  double max_drawdown_pct = 11;
  crewless.common.ApprovalState approval_state = 12;
}
```

### 5.3 `policy.proto`

```proto
syntax = "proto3";
package crewless.policy;
import "common.proto";

message ToolGrant {
  crewless.common.Meta meta = 1;
  string grant_id = 2;
  string subject_agent_id = 3;
  string tool_name = 4;
  crewless.common.GrantEffect effect = 5;
  repeated string allowed_actions = 6;
  repeated string allowed_resources = 7;
  int64 expires_at_ms = 8;
}

message RepoGrant {
  crewless.common.Meta meta = 1;
  string grant_id = 2;
  string subject_agent_id = 3;
  string repo_id = 4;
  string branch_pattern = 5;
  bool read_allowed = 6;
  bool write_allowed = 7;
  bool merge_allowed = 8;
  bool tag_allowed = 9;
}

message SecretGrant {
  crewless.common.Meta meta = 1;
  string grant_id = 2;
  string subject_agent_id = 3;
  string secret_ref = 4;
  string purpose = 5;
  int64 expires_at_ms = 6;
}

message ServiceGrant {
  crewless.common.Meta meta = 1;
  string grant_id = 2;
  string subject_agent_id = 3;
  string service_name = 4;
  repeated string allowed_methods = 5;
  repeated string resource_scopes = 6;
  int64 expires_at_ms = 7;
}

message CapitalAuthorityGrant {
  crewless.common.Meta meta = 1;
  string grant_id = 2;
  string department_id = 3;
  string planner_agent_id = 4;
  repeated string chains = 5;
  repeated string protocols = 6;
  repeated string instruments = 7;
  double max_notional_usd = 8;
  double max_daily_notional_usd = 9;
  double max_leverage = 10;
  bool allow_reduce_only = 11;
  bool live_enabled = 12;
  int64 expires_at_ms = 13;
}
```

### 5.4 `decisioning.proto`

```proto
syntax = "proto3";
package crewless.decisioning;
import "common.proto";

message StrategyValidationReport {
  crewless.common.Meta meta = 1;
  string strategy_id = 2;
  string strategy_version = 3;
  string benchmark = 4;
  double sharpe = 5;
  double sortino = 6;
  double calmar = 7;
  double max_drawdown_pct = 8;
  double profit_factor = 9;
  double win_rate = 10;
  double turnover = 11;
  repeated string bias_checks_passed = 12;
  repeated string open_issues = 13;
  crewless.common.ApprovalState approval_state = 14;
}

message ResearchPacket {
  crewless.common.Meta meta = 1;
  string strategy_id = 2;
  string asset_universe = 3;
  repeated string evidence_refs = 4;
  repeated string findings = 5;
  repeated string risks = 6;
}

message TradeIntent {
  crewless.common.Meta meta = 1;
  string strategy_id = 2;
  string instrument = 3;
  string action = 4;
  double confidence = 5;
  double target_notional_pct = 6;
  double max_slippage_bps = 7;
  string rationale = 8;
  repeated string required_conditions = 9;
}

message RiskReview {
  crewless.common.Meta meta = 1;
  string committee_result = 2;
  double approved_notional_pct = 3;
  repeated string objections = 4;
  repeated string constraints = 5;
}

message ExecutionPlan {
  crewless.common.Meta meta = 1;
  string plan_id = 2;
  string venue = 3;
  string chain = 4;
  string protocol = 5;
  string instrument = 6;
  string side = 7;
  double notional_usd = 8;
  double leverage = 9;
  uint32 max_slippage_bps = 10;
  bool reduce_only = 11;
  string tif = 12;
  string idempotency_key = 13;
}
```

### 5.5 `audit.proto`

```proto
syntax = "proto3";
package crewless.audit;
import "common.proto";

message DecisionTraceRef {
  crewless.common.Meta meta = 1;
  string trace_id = 2;
  string cycle_id = 3;
  string strategy_id = 4;
  string strategy_version = 5;
  string repo_commit = 6;
  string prompt_policy_version = 7;
  repeated string artifact_ids = 8;
}

message SecurityFinding {
  crewless.common.Meta meta = 1;
  string finding_id = 2;
  string severity = 3;
  string category = 4;
  string subject_id = 5;
  string description = 6;
  string remediation = 7;
  crewless.common.ApprovalState status = 8;
}
```

---

## 6. JSON-isomorphic examples

### 6.1 `CapitalAuthorityGrant`

```json
{
  "meta": {
    "event_id": "evt_01J...",
    "correlation_id": "corr_01J...",
    "trace_id": "tr_01J...",
    "cycle_id": "",
    "department_id": "dept_momentum_btc",
    "agent_id": "planner_btc_momo_01",
    "actor_id": "hermes",
    "policy_version": "policy.v3",
    "schema_version": "1.0.0",
    "created_at_ms": 1776321000000
  },
  "grant_id": "cag_01J...",
  "department_id": "dept_momentum_btc",
  "planner_agent_id": "planner_btc_momo_01",
  "chains": ["hyperliquid"],
  "protocols": ["hyperliquid_perps"],
  "instruments": ["BTC-PERP"],
  "max_notional_usd": 25000,
  "max_daily_notional_usd": 75000,
  "max_leverage": 2,
  "allow_reduce_only": true,
  "live_enabled": false,
  "expires_at_ms": 1778913000000
}
```

### 6.2 `ExecutionPlan`

```json
{
  "meta": {
    "event_id": "evt_01J...",
    "correlation_id": "corr_01J...",
    "trace_id": "tr_01J...",
    "cycle_id": "cyc_01J...",
    "department_id": "dept_momentum_btc",
    "agent_id": "planner_btc_momo_01",
    "actor_id": "planner_btc_momo_01",
    "policy_version": "policy.v3",
    "schema_version": "1.0.0",
    "created_at_ms": 1776321300000
  },
  "plan_id": "plan_01J...",
  "venue": "hyperliquid",
  "chain": "hyperliquid",
  "protocol": "hyperliquid_perps",
  "instrument": "BTC-PERP",
  "side": "LONG",
  "notional_usd": 12000,
  "leverage": 1.5,
  "max_slippage_bps": 12,
  "reduce_only": false,
  "tif": "IOC",
  "idempotency_key": "cyc_01J...:slice_01"
}
```

---

## 7. `agents.yaml` pattern

Use `agents.yaml` as the machine-readable declaration of identity, permissions intent, and environment bindings. It is not the final enforcement point, but it should map 1:1 to generated policy bundles.

```yaml
version: 1
organization: crewless-capital
policy_version: policy.v3

environments:
  research:
    live_enabled: false
  paper:
    live_enabled: false
  shadow:
    live_enabled: false
  limited:
    live_enabled: true
  full:
    live_enabled: true

departments:
  - id: dept_momentum_btc
    name: BTC Momentum
    charter_ref: charters/dept_momentum_btc.yaml
    environment: shadow
    allowed_strategy_classes: [momentum, trend]
    allowed_chains: [hyperliquid]
    allowed_protocols: [hyperliquid_perps]
    allowed_instruments: [BTC-PERP]
    max_gross_exposure_pct: 0.10
    max_daily_turnover_pct: 0.30
    max_drawdown_pct: 0.03

agents:
  - id: hermes
    layer: supervisor
    role: firm_supervisor
    department_id: executive
    soul_ref: souls/hermes/SOUL.md
    prompt_ref: prompts/hermes/system_prompt.md
    env: full
    repos: [governance-config, policy-registry]
    peers: [orchestrator_api, security_controller]
    tools:
      - name: governance_api
        actions: [read, approve, reject, freeze]
      - name: observability_api
        actions: [read]
    secrets: []
    restrictions:
      no_execution_tools: true
      no_repo_merge_outside_governance: true

  - id: research_btc_momo_01
    layer: worker
    role: research_agent
    department_id: dept_momentum_btc
    soul_ref: souls/research_btc_momo_01/SOUL.md
    prompt_ref: prompts/research/system_prompt.md
    env: research
    repos: [strategy-btc-momentum]
    peers: [debate_bull_btc_01, debate_bear_btc_01]
    tools:
      - name: market_data_api
        actions: [read]
      - name: research_rag
        actions: [read]
      - name: git_adapter
        actions: [read, write_branch, open_pr]
    secrets:
      - ref: secret/data/market_readonly
    restrictions:
      no_execution_tools: true
      no_live_environment: true

  - id: planner_btc_momo_01
    layer: department
    role: execution_planner
    department_id: dept_momentum_btc
    soul_ref: souls/planner_btc_momo_01/SOUL.md
    prompt_ref: prompts/planner/system_prompt.md
    env: shadow
    repos: [strategy-btc-momentum]
    peers: [risk_dept_btc_momo_01, orchestrator_api]
    tools:
      - name: plan_writer
        actions: [create_execution_plan]
      - name: trace_api
        actions: [read, write]
    secrets: []
    restrictions:
      no_wallet_access: true
      no_raw_order_submission: true
      must_emit_typed_execution_plan: true
```

### 7.1 Generation rule

`agents.yaml` should be compiled into:
- OpenClaw gateway policy,
- K8s service accounts / workload identities,
- repo ACL requests,
- Vault policy documents,
- NATS subject publish/subscribe permissions,
- runtime admission policies.

---

## 8. Repo / secret / ACL matrix by layer

### 8.1 Matrix

| Layer | Repo access | Secret access | Filesystem | Network/service access | Capital authority |
|---|---|---|---|---|---|
| Layer 0 Creator | Governance repos read/write, signed release tags | Break-glass only, human-held | Full admin workstation only | Executive console, policy registry | Human approval only, no raw key export |
| Layer 1 Hermes | Governance + policy repos, no strategy self-merge | Read policy metadata, no venue signing secrets | Read-only except incident bundles | Observability, governance API, security controller | None directly |
| Layer 2 Workers | Strategy repos branch-only, PR-only | Narrow data/API creds per task | Per-agent workspace, isolated user/container | RAG, market data, research services, repo adapter | None |
| Layer 3 Department staff | Department repos read, limited write to generated artifacts | Department-scoped readonly and planning secrets | Department workspace only | Orchestrator, trace API, plan writer | Plan authoring only |
| Firm Risk Core | No prompt repos needed, policy registry read | Risk config, market state access | Minimal immutable runtime image | Decision stream consume, approval emit | Approve/reject only |
| Wallet Policy Service | No agent repos | Vault-bound signer policy refs, no human-readable export | Hardened service FS only | Execution router, chain adapters, reconciler | Bounded grant issuance only |
| Execution Router | No strategy repo write | Venue auth material or signer references only | Ephemeral runtime only | Chain/protocol adapters only | Submit approved staged requests |
| Reconciler | No strategy repo write | Read-only chain/RPC creds | Ephemeral runtime only | Chain readers, trace store, security stream | None |

### 8.2 Practical ACL rules

- Research agents: branch-only, PR-only, no merge.
- Department agents: cannot modify `policies/`, `wallet/`, `infra/prod/`, or `governance/`.
- Hermes: may freeze or revoke, not deploy execution code.
- Execution services: cannot read prompt repos or arbitrary workspace files.
- Wallet policy and execution router run in separate namespaces and service accounts.

---

## 9. NATS authz model

Map identity to NATS subject permissions.

### 9.1 Example rules

- Research agents:
  - publish: `decision.research_packet.created`, `agent.heartbeat.*`
  - subscribe: `governance.agent.<agent_id>.>`, `obs.control.<agent_id>.>`
  - deny: `execution.>`, `wallet.>`, `security.admin.>`

- Planners:
  - publish: `decision.trade_intent.created`, `decision.execution_plan.created`
  - subscribe: `decision.research_packet.created`, `decision.risk_review.*`
  - deny: `execution.router.submitted`, `wallet.grant.issued`

- Firm Risk Core:
  - subscribe: `decision.execution_plan.created`
  - publish: `execution.plan.approved`, `execution.plan.rejected`

- Execution Router:
  - subscribe: `execution.plan.approved`
  - publish: `execution.router.submitted`, `execution.fill.reported`

This makes the event bus part of the hard boundary, not just transport plumbing.

---

## 10. Command flow with JetStream

### 10.1 Research to execution path

1. Research agent publishes `decision.research_packet.created`.
2. Debate/trader services publish `decision.trade_intent.created`.
3. Dept risk publishes `decision.risk_review.created`.
4. Planner publishes `decision.execution_plan.created`.
5. Firm Risk Core consumes plan, validates grants/policy/exposure, persists checkpoint, publishes either:
   - `execution.plan.approved`, or
   - `execution.plan.rejected`.
6. Wallet Policy Service consumes approved plan, binds grant and signer scope, emits `execution.wallet_scope.bound`.
7. Execution Router consumes both approval and wallet scope, submits transactions, emits `execution.router.submitted` and `execution.fill.reported`.
8. Reconciler consumes fill + chain state, emits `reconcile.position.confirmed` or `reconcile.position.drift_detected`.
9. DecisionTrace is finalized with all artifact refs.

### 10.2 Idempotency pattern

Use `idempotency_key = cycle_id + plan_id + slice_index` across:
- risk approval,
- wallet grant binding,
- router submission,
- reconciler finalization.

Because JetStream consumers are at-least-once, every downstream stage must be replay-safe.[web:51][web:52]

---

## 11. Security controller integration

### 11.1 ClawSec-style controls

Use ClawSec or equivalent for:
- `SOUL.md` integrity drift;
- prompt-policy version attestation;
- tool manifest validation;
- unsafe capability diff detection;
- CI blocking on unauthorized grant expansion;
- detection of disallowed files or unsafe runtime configs.

### 11.2 Clawvisor-style controls

Use Clawvisor or equivalent for:
- runtime posture dashboards;
- approval workflows;
- agent suspension;
- environment promotion approvals;
- cross-department anomaly alerting;
- incident-mode force pause.

### 11.3 Required security events

Publish to `security.>`:
- `security.integrity.scan_passed`
- `security.integrity.scan_failed`
- `security.grant.expansion_requested`
- `security.grant.expansion_rejected`
- `security.runtime.drift_detected`
- `security.agent.quarantined`

---

## 12. Department formation workflow

### 12.1 Lifecycle

1. **Incubation proposal**
   - research agents create `DepartmentProposal` artifact.
2. **Validation package**
   - include validation report, repo refs, risk notes, staffing pattern.
3. **Security review**
   - verify grants, prompt integrity, repo ACL map, NATS permissions.
4. **Charter issuance**
   - signed `DepartmentCharter` enters policy registry.
5. **Staff instantiation**
   - create analysts, debate roles, trader/planner, dept-risk roles.
6. **Shadow activation**
   - no live wallet scope; emit plans only.
7. **Limited capital activation**
   - bounded capital grant issued by Wallet Policy Service.

### 12.2 Department proposal JSON

```json
{
  "proposal_id": "deptprop_01J...",
  "name": "BTC Momentum",
  "strategy_family": "momentum",
  "dept_owner": "momentum_department",
  "target_instruments": ["BTC-PERP"],
  "target_chains": ["hyperliquid"],
  "required_roles": [
    "analyst_technical",
    "analyst_onchain",
    "bull_researcher",
    "bear_researcher",
    "trader",
    "dept_risk"
  ],
  "validation_report_ref": "valrep_01J...",
  "repo_refs": ["strategy-btc-momentum"],
  "recommended_environment": "SHADOW_ALLOCATED",
  "requested_caps": {
    "max_notional_usd": 25000,
    "max_leverage": 2,
    "max_daily_turnover_pct": 0.3
  }
}
```

---

## 13. Implementation defaults

### 13.1 Language split

- TypeScript or Go:
  - Orchestrator API
  - Firm Risk Core
  - Wallet Policy Service
  - NATS consumers/publishers for critical stages

- Python:
  - research and departmental agents
  - backtests and evaluation jobs
  - adapter-heavy integrations

### 13.2 State systems

- Postgres: system of record for policies, approvals, DecisionTrace index, governance.
- TimescaleDB/Postgres extension: market/event series if needed.
- Object store: large artifacts, reports, backtests.
- JetStream: durable event transport and replay.
- Vault/secret manager: all credentials and signer indirection.

---

## 14. Recommended repo layout

```text
crewless-capital/
  proto/
    common.proto
    identity.proto
    policy.proto
    decisioning.proto
    audit.proto
  config/
    agents.yaml
    departments/
    policy-bundles/
    nats/
  services/
    orchestrator-api/
    firm-risk-core/
    wallet-policy-service/
    execution-router/
    reconciler/
    security-controller/
  agents/
    dept-momentum/
    dept-meanreversion/
    incubation-lab/
  governance/
    SOULs/
    prompts/
    charters/
    promotion-records/
  docs/
    SECURITY_ARCHITECTURE_BRIEF.md
```

---

## 15. Immediate next steps

1. Freeze the canonical protobuf schema set and generate TS/Python clients.
2. Compile `agents.yaml` into a policy bundle generator.
3. Stand up JetStream with streams, durable consumers, and subject ACLs.
4. Implement the minimal path:
   - ResearchPacket
   - TradeIntent
   - ExecutionPlan
   - Risk approval/reject
   - DecisionTraceRef
5. Add CI checks for:
   - unauthorized grant expansion,
   - repo ACL violations,
   - NATS subject overreach,
   - prompt/spec drift.
6. Pilot one department only first: BTC momentum on HyperLiquid, shadow mode.

## 16. Non-negotiable invariants

- Agents propose; deterministic services approve.
- No wallet key material enters any agent runtime.
- No direct publish permission to `execution.plan.approved` outside Firm Risk Core.
- No direct subscribe permission to wallet scope subjects outside Wallet Policy Service and Execution Router.
- No department promotion without signed charter and validation report.
- Every execution-affecting event must reference a DecisionTrace.
- Every critical service must tolerate replay and duplicate delivery from JetStream.
