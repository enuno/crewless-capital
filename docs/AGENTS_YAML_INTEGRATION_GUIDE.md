# `agents.yaml` Integration Guide — CrewAI & LangChain

**Repo:** https://github.com/enuno/crewless-capital
**Spec version:** 2.1.0-draft
**Status:** Research / Paper trading target
**Applies to:** `config/agents.yaml`, `departments/*/mandate.yaml`, `packages/prompt-policies/`

---

## Objective

This guide specifies how `agents.yaml` — the machine-readable identity and capability declaration defined in the Security Architecture Brief — integrates with CrewAI and LangChain as the department agent runtime in Crewless Capital. It maps each field in `agents.yaml` to CrewAI and LangChain primitives, provides loader patterns, and establishes the policy enforcement layer that must wrap both frameworks before they are permitted to act within the firm's capability plane.

**Key invariant:** CrewAI and LangChain are orchestration and worker frameworks. They have no execution authority. Every typed artifact they produce passes through the Firm Risk Core and Wallet Policy Service before any on-chain or capital action occurs.

---

## 1. Where agents.yaml lives in the repo

```
crewless-capital/
  config/
    agents.yaml                        ← Firm-level agent registry (reviewed here)
    departments/
      momentum/agents.yaml             ← Department-scoped agent overrides
      treasury-yield/agents.yaml
  departments/
    runtime-host/                      ← Generic department runtime (loads agents.yaml)
    momentum/
      mandate.yaml                     ← Department constraints used to gate tool grants
```

The loader runs at department startup and must:
1. Validate the file against the canonical JSON schema (generated from `identity.proto`).
2. Resolve each agent's `soul_ref`, `prompt_ref`, and `tools` against the Policy Registry.
3. Build and inject CrewAI `Agent` or LangChain `RunnableAgent` objects only if all grants are valid.
4. Register each agent with the NATS JetStream subject ACL before allowing it to publish.

---

## 2. Canonical agents.yaml structure

```yaml
version: 1
organization: crewless-capital
policy_version: policy.v3

environments:
  research:
    live_enabled: false
    allow_tool_classes: [data_read, repo_write, rag_read]
  paper:
    live_enabled: false
    allow_tool_classes: [data_read, repo_write, rag_read, plan_write]
  shadow:
    live_enabled: false
    allow_tool_classes: [data_read, repo_read, rag_read, plan_write, trace_write]
  limited:
    live_enabled: true
    allow_tool_classes: [data_read, repo_read, rag_read, plan_write, trace_write]
  full:
    live_enabled: true
    allow_tool_classes: [data_read, repo_read, rag_read, plan_write, trace_write]

departments:
  - id: dept_momentum
    name: Momentum / Trend
    charter_ref: config/departments/momentum/charter.yaml
    mandate_ref: departments/momentum/mandate.yaml
    environment: shadow
    allowed_strategy_classes: [momentum, trend]
    allowed_chains: [hyperliquid, arbitrum, base]
    allowed_protocols: [hyperliquid_perps, gmx, uniswap_v3]
    allowed_instruments: [BTC-PERP, ETH-PERP]
    max_gross_exposure_pct: 0.10
    max_daily_turnover_pct: 0.30
    max_drawdown_pct: 0.03

agents:
  # ── Executive layer ────────────────────────────────────────────────
  - id: hermes
    layer: supervisor
    role: firm_supervisor
    department_id: executive
    framework: crewai                  # crewai | langchain | deterministic
    soul_ref: governance/SOULs/hermes/SOUL.md
    prompt_ref: packages/prompt-policies/hermes/system_prompt.md
    model_routing_tier: mid
    environment: full
    repos:
      - id: governance-config
        read: true
        write: true
        merge: false
    peers: [orchestrator_api, security_controller]
    tools:
      - name: governance_api
        tool_class: governance
        actions: [read, approve, reject, freeze]
      - name: observability_api
        tool_class: data_read
        actions: [read]
    secrets: []
    nats:
      publish: [governance.approval.>, governance.freeze.>, agent.heartbeat.hermes]
      subscribe: [security.>.>, agent.lifecycle.>, obs.control.hermes.>]
      deny: [execution.>, wallet.>]
    restrictions:
      no_execution_tools: true
      no_repo_merge_outside_governance: true
      no_wallet_access: true

  # ── Research layer (Incubation / Research environment) ────────────
  - id: research_momentum_01
    layer: worker
    role: research_agent
    department_id: dept_momentum
    framework: crewai
    soul_ref: governance/SOULs/research/SOUL.md
    prompt_ref: packages/prompt-policies/research/system_prompt.md
    model_routing_tier: fast
    environment: research
    repos:
      - id: strategy-momentum
        read: true
        write: true        # branch-only; merge blocked by repo ACL
        merge: false
        branch_pattern: "research/*"
    peers: [debate_bull_momentum_01, debate_bear_momentum_01]
    tools:
      - name: market_data_api
        tool_class: data_read
        actions: [read]
      - name: research_rag
        tool_class: rag_read
        actions: [read]
      - name: onchain_reader
        tool_class: data_read
        actions: [read]
      - name: git_adapter
        tool_class: repo_write
        actions: [read, write_branch, open_pr]
    secrets:
      - ref: secret/data/market_readonly
        purpose: market data API read access
        expires_at_ms: 0   # managed by secret manager; 0 = defer to manager TTL
    nats:
      publish: [agent.output.research_packet, agent.heartbeat.research_momentum_01]
      subscribe: [governance.agent.research_momentum_01.>, obs.control.research_momentum_01.>]
      deny: [execution.>, wallet.>, decision.execution_plan.>]
    restrictions:
      no_execution_tools: true
      no_live_environment: true
      no_wallet_access: true
      output_schema: ResearchPacket

  # ── Department analyst ─────────────────────────────────────────────
  - id: analyst_technical_momentum
    layer: department
    role: technical_analyst
    department_id: dept_momentum
    framework: langchain
    soul_ref: governance/SOULs/dept_momentum/analyst_technical/SOUL.md
    prompt_ref: packages/prompt-policies/analysts/technical/system_prompt.md
    model_routing_tier: mid
    environment: shadow
    repos:
      - id: strategy-momentum
        read: true
        write: false
        merge: false
    peers: [analyst_onchain_momentum, debate_bull_momentum_01]
    tools:
      - name: market_data_api
        tool_class: data_read
        actions: [read]
      - name: technical_indicators_lib
        tool_class: data_read
        actions: [compute]
      - name: trace_api
        tool_class: trace_write
        actions: [write]
    secrets:
      - ref: secret/data/market_readonly
        purpose: price series and indicators
    nats:
      publish: [agent.output.analyst_report, agent.heartbeat.analyst_technical_momentum]
      subscribe: [agent.output.research_packet, obs.control.analyst_technical_momentum.>]
      deny: [execution.>, wallet.>, governance.approval.>]
    restrictions:
      no_execution_tools: true
      no_wallet_access: true
      output_schema: AnalystScore

  # ── Department execution planner ───────────────────────────────────
  - id: planner_momentum_01
    layer: department
    role: execution_planner
    department_id: dept_momentum
    framework: crewai
    soul_ref: governance/SOULs/dept_momentum/planner/SOUL.md
    prompt_ref: packages/prompt-policies/planner/system_prompt.md
    model_routing_tier: strong_reasoning
    environment: shadow
    repos:
      - id: strategy-momentum
        read: true
        write: false
        merge: false
    peers: [dept_risk_momentum_01, orchestrator_api]
    tools:
      - name: plan_writer
        tool_class: plan_write
        actions: [create_execution_plan]
      - name: trace_api
        tool_class: trace_write
        actions: [read, write]
    secrets: []
    nats:
      publish:
        - decision.trade_intent.created
        - decision.execution_plan.created
        - agent.heartbeat.planner_momentum_01
      subscribe:
        - decision.risk_review.created
        - agent.output.trader_synthesis
        - obs.control.planner_momentum_01.>
      deny: [execution.router.>, wallet.>, execution.plan.approved]
    restrictions:
      no_wallet_access: true
      no_raw_order_submission: true
      must_emit_typed_execution_plan: true
      output_schema: ExecutionPlan
```

---

## 3. Schema validation

All `agents.yaml` files must be validated against the canonical JSON schema before use. Generate the schema from `identity.proto` via `buf` or `protoc-gen-jsonschema`.

```bash
# Generate JSON schema from proto
buf generate --template buf.gen.yaml proto/identity.proto

# Validate at load time (Python)
pip install jsonschema pyyaml

# departments/runtime-host/loader.py (excerpt below)
```

```python
# departments/runtime-host/loader.py
import yaml, json
from jsonschema import validate, ValidationError
from pathlib import Path

SCHEMA_PATH = Path("packages/schemas/agents_schema.json")
AGENTS_PATH = Path("config/agents.yaml")

def load_and_validate() -> dict:
    schema = json.loads(SCHEMA_PATH.read_text())
    raw = yaml.safe_load(AGENTS_PATH.read_text())
    try:
        validate(instance=raw, schema=schema)
    except ValidationError as e:
        raise RuntimeError(f"agents.yaml validation failed: {e.message}") from e
    return raw
```

---

## 4. CrewAI integration

CrewAI maps onto Crewless Capital's department pipeline naturally: each specialist produces a typed artifact (a CrewAI `Task` output), the crew runs sequential or parallel tasks, and the final planner emits an `ExecutionPlan`.

### 4.1 Framework mapping

| agents.yaml field | CrewAI concept | Notes |
|---|---|---|
| `id` | `agent.id` (set via `Agent(name=...)`) | Used as JetStream publisher identity |
| `role` | `Agent(role=...)` | Loaded from spec, not hardcoded |
| `soul_ref` / `prompt_ref` | `Agent(backstory=..., goal=...)` | Read from file; never interpolated at runtime |
| `tools[].name` | `Agent(tools=[...])` | Tools built from `tool_class` grants only |
| `model_routing_tier` | `Agent(llm=...)` | Resolved via `packages/model-routing/` |
| `output_schema` | `Task(output_pydantic=...)` | Forces typed output; parse failure = retry |
| `restrictions.no_execution_tools` | Tool grant filter | Enforced by loader before crew is built |
| `environment` | Crew-level guard check | Crew.kickoff blocked if env mismatch |

### 4.2 Agent factory

```python
# departments/runtime-host/crewai_factory.py
from crewai import Agent, Task, Crew, Process
from pathlib import Path
from typing import Any
import yaml

from .loader import load_and_validate
from .tool_registry import build_tools_for_agent   # see §4.3
from .model_router import resolve_llm              # see §4.4
from .nats_publisher import NATSPublisher

FIRM_ENV = "shadow"   # injected from env var CREWLESS_ENV at startup


def build_agent(spec: dict, dept_id: str) -> Agent:
    assert spec["environment"] == FIRM_ENV, (
        f"Agent {spec['id']} declared for env '{spec['environment']}' "
        f"but firm is running '{FIRM_ENV}'. Refusing to instantiate."
    )
    assert spec["department_id"] == dept_id, (
        f"Agent {spec['id']} belongs to dept '{spec['department_id']}', "
        f"not '{dept_id}'. Cross-department instantiation blocked."
    )

    soul = Path(spec["soul_ref"]).read_text()
    prompt = Path(spec["prompt_ref"]).read_text()
    tools = build_tools_for_agent(spec)     # enforces tool_class allowlist
    llm = resolve_llm(spec["model_routing_tier"])

    return Agent(
        id=spec["id"],
        role=spec["role"],
        goal=prompt,        # system_prompt.md is the goal/instruction
        backstory=soul,     # SOUL.md is the backstory / identity
        tools=tools,
        llm=llm,
        verbose=False,
        allow_delegation=False,   # never allow cross-agent delegation without spec
        max_iter=5,
    )


def build_department_crew(dept_id: str, cycle_id: str) -> Crew:
    config = load_and_validate()
    dept_agents_spec = [
        a for a in config["agents"]
        if a["department_id"] == dept_id
    ]

    agents = {s["id"]: build_agent(s, dept_id) for s in dept_agents_spec}

    # Build tasks in pipeline order — outputs are typed pydantic models
    from .tasks import build_department_tasks
    tasks = build_department_tasks(agents, cycle_id)

    return Crew(
        agents=list(agents.values()),
        tasks=tasks,
        process=Process.sequential,
        verbose=False,
    )
```

### 4.3 Tool factory with grant enforcement

```python
# departments/runtime-host/tool_registry.py
from crewai_tools import tool
from typing import Callable
import httpx

# Map from tool_class → allowed tool builders
TOOL_CLASS_REGISTRY: dict[str, Callable] = {
    "data_read":    lambda cfg: build_market_data_tool(cfg),
    "rag_read":     lambda cfg: build_rag_tool(cfg),
    "repo_write":   lambda cfg: build_git_tool(cfg),
    "plan_write":   lambda cfg: build_plan_writer_tool(cfg),
    "trace_write":  lambda cfg: build_trace_tool(cfg),
    # "execution" and "wallet" tool classes are never registered here
    # attempting to add them raises a hard ConfigurationError
}

BLOCKED_TOOL_CLASSES = {"execution", "wallet", "signing", "broadcast"}


def build_tools_for_agent(agent_spec: dict) -> list:
    tools = []
    for tool_spec in agent_spec.get("tools", []):
        tc = tool_spec["tool_class"]
        if tc in BLOCKED_TOOL_CLASSES:
            raise ConfigurationError(
                f"Agent '{agent_spec['id']}' attempted to register tool class "
                f"'{tc}' which is permanently blocked for all agent runtimes."
            )
        if tc not in TOOL_CLASS_REGISTRY:
            raise ConfigurationError(
                f"Unknown tool_class '{tc}' for agent '{agent_spec['id']}'."
            )
        if tc not in get_env_allowed_classes(agent_spec["environment"]):
            raise ConfigurationError(
                f"Tool class '{tc}' not permitted in environment "
                f"'{agent_spec['environment']}' for agent '{agent_spec['id']}'."
            )
        tools.append(TOOL_CLASS_REGISTRY[tc](tool_spec))
    return tools
```

### 4.4 Model router shim

```python
# departments/runtime-host/model_router.py
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
import os

# Loaded from packages/model-routing/routing_table.yaml
ROUTING_TABLE = {
    "fast":             {"primary": "gpt-4o-mini",  "failover": "claude-haiku-3-5"},
    "mid":              {"primary": "gpt-4o",        "failover": "claude-sonnet-4-5"},
    "strong_reasoning": {"primary": "o3",            "failover": "claude-opus-4"},
}

def resolve_llm(tier: str):
    entry = ROUTING_TABLE[tier]
    model = entry["primary"]
    # simple provider check; production version polls model health table
    if model.startswith("gpt") or model.startswith("o"):
        return ChatOpenAI(model=model, temperature=0)
    if model.startswith("claude"):
        return ChatAnthropic(model=model, temperature=0)
    raise ValueError(f"Unknown model '{model}' for tier '{tier}'")
```

### 4.5 Typed task outputs

All tasks must declare an `output_pydantic` model. If the LLM fails to emit a valid schema, the task retries up to `max_retries` and then emits a `HOLD_FLAT` cycle via the Orchestrator API.

```python
# departments/runtime-host/tasks.py
from crewai import Task
from pydantic import BaseModel
from typing import List, Optional


class AnalystScoreOutput(BaseModel):
    agent_id: str
    score: float
    confidence: float
    key_points: List[str]
    evidence_refs: List[str]


class ResearchPacketOutput(BaseModel):
    cycle_id: str
    strategy_id: str
    asset_universe: str
    evidence_refs: List[str]
    findings: List[str]
    risks: List[str]


class TradeIntentOutput(BaseModel):
    cycle_id: str
    strategy_id: str
    instrument: str
    action: str         # LONG | SHORT | FLAT
    confidence: float
    target_notional_pct: float
    max_slippage_bps: float
    rationale: str
    required_conditions: List[str]


class ExecutionPlanOutput(BaseModel):
    cycle_id: str
    plan_id: str
    venue: str
    chain: str
    protocol: str
    instrument: str
    side: str
    notional_usd: float
    leverage: float
    max_slippage_bps: float
    reduce_only: bool
    tif: str
    idempotency_key: str


def build_department_tasks(agents: dict, cycle_id: str) -> list:
    analyst_task = Task(
        description=f"Produce a technical analyst score for cycle {cycle_id}. "
                    "Output must conform to AnalystScoreOutput exactly.",
        expected_output="AnalystScoreOutput JSON",
        agent=agents["analyst_technical_momentum"],
        output_pydantic=AnalystScoreOutput,
    )

    planner_task = Task(
        description=f"Given the research packet and risk review for cycle {cycle_id}, "
                    "produce an ExecutionPlan. Output must conform to ExecutionPlanOutput exactly. "
                    "Do not include any wallet access, signing, or submission logic.",
        expected_output="ExecutionPlanOutput JSON",
        agent=agents["planner_momentum_01"],
        output_pydantic=ExecutionPlanOutput,
        context=[analyst_task],
    )

    return [analyst_task, planner_task]
```

---

## 5. LangChain integration

LangChain is used for analyst-tier agents in Crewless Capital where fine-grained chain control and tool calling are preferable to CrewAI's crew abstraction. The same `agents.yaml` drives both frameworks via the shared loader.

### 5.1 Framework mapping

| agents.yaml field | LangChain concept | Notes |
|---|---|---|
| `id` | `RunnablePassthrough` metadata tag | Injected into every runnable invocation |
| `role` / `soul_ref` | `SystemMessagePromptTemplate` | Loaded from file; never hot-patched |
| `prompt_ref` | `HumanMessagePromptTemplate` | The task instruction |
| `tools[].name` | `tool` decorated functions → `bind_tools()` | Filtered by `tool_class` grant |
| `model_routing_tier` | `ChatOpenAI` / `ChatAnthropic` init | Resolved via model_router |
| `output_schema` | `.with_structured_output(PydanticModel)` | Forces typed response; retried on failure |
| `restrictions` | Pre-runnable validation guard | Raises if banned tools/env attempted |

### 5.2 Agent factory

```python
# departments/runtime-host/langchain_factory.py
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables import RunnableLambda
from langchain_core.messages import SystemMessage
from langgraph.prebuilt import create_react_agent
from pathlib import Path

from .loader import load_and_validate
from .tool_registry import build_tools_for_agent
from .model_router import resolve_llm
from .nats_publisher import NATSPublisher

FIRM_ENV = "shadow"


def build_langchain_agent(agent_id: str, dept_id: str):
    config = load_and_validate()
    spec = next(
        (a for a in config["agents"] if a["id"] == agent_id),
        None
    )
    assert spec is not None, f"Agent '{agent_id}' not found in agents.yaml"
    assert spec["environment"] == FIRM_ENV, (
        f"Agent '{agent_id}' env mismatch: spec='{spec['environment']}' firm='{FIRM_ENV}'"
    )
    assert spec["department_id"] == dept_id, (
        f"Agent '{agent_id}' dept mismatch"
    )

    soul = Path(spec["soul_ref"]).read_text()
    prompt = Path(spec["prompt_ref"]).read_text()
    tools = build_tools_for_agent(spec)
    llm = resolve_llm(spec["model_routing_tier"])

    system_prompt = f"{soul}\n\n---\n\n{prompt}"
    model_with_tools = llm.bind_tools(tools)

    # Use LangGraph ReAct agent for tool-calling agents
    agent = create_react_agent(
        model=model_with_tools,
        tools=tools,
        state_modifier=system_prompt,
    )

    return agent, spec


def run_langchain_agent(agent_id: str, dept_id: str, input_payload: dict,
                        output_model, cycle_id: str):
    agent, spec = build_langchain_agent(agent_id, dept_id)
    model_with_structured = resolve_llm(
        spec["model_routing_tier"]
    ).with_structured_output(output_model)

    result = model_with_structured.invoke([
        SystemMessage(content=Path(spec["soul_ref"]).read_text()),
        {"role": "user", "content": str(input_payload)},
    ])

    # Publish typed artifact to NATS
    publisher = NATSPublisher()
    nats_subjects = spec["nats"]["publish"]
    publisher.publish(
        subject=nats_subjects[0],
        payload=result.model_dump(),
        agent_id=agent_id,
        cycle_id=cycle_id,
    )
    return result
```

### 5.3 Structured output enforcement

```python
# departments/runtime-host/structured_runner.py
from pydantic import ValidationError
from .nats_publisher import NATSPublisher
from .orchestrator_client import emit_hold_flat

MAX_RETRIES = 2

def run_with_structured_output(llm, messages: list, output_model,
                                agent_id: str, cycle_id: str):
    model = llm.with_structured_output(output_model, strict=True)
    for attempt in range(MAX_RETRIES + 1):
        try:
            result = model.invoke(messages)
            return result
        except (ValidationError, Exception) as e:
            if attempt == MAX_RETRIES:
                emit_hold_flat(
                    cycle_id=cycle_id,
                    reason=f"Agent '{agent_id}' failed structured output "
                           f"after {MAX_RETRIES+1} attempts: {e}"
                )
                return None
    return None
```

---

## 6. NATS JetStream wiring

Every agent — regardless of framework — must publish to and consume from NATS JetStream only via the subjects declared in `agents.yaml`. The `NATSPublisher` enforces the subject allowlist at runtime.

### 6.1 NATSPublisher

```python
# departments/runtime-host/nats_publisher.py
import nats
import json
from typing import Any

class NATSPublisher:
    def __init__(self):
        self._nc = None
        self._js = None

    async def connect(self, nats_url: str = "nats://localhost:4222"):
        self._nc = await nats.connect(nats_url)
        self._js = self._nc.jetstream()

    async def publish(self, subject: str, payload: dict,
                      agent_id: str, cycle_id: str,
                      allowed_subjects: list[str]):
        # Enforce subject allowlist from agents.yaml
        if not any(
            subject == s or subject.startswith(s.rstrip(">"))
            for s in allowed_subjects
        ):
            raise PermissionError(
                f"Agent '{agent_id}' attempted to publish to '{subject}' "
                f"which is not in its declared publish allowlist."
            )
        envelope = {
            "agent_id": agent_id,
            "cycle_id": cycle_id,
            "payload": payload,
        }
        ack = await self._js.publish(
            subject,
            json.dumps(envelope).encode(),
            headers={"Nats-Msg-Id": f"{agent_id}:{cycle_id}:{subject}"}  # dedup key
        )
        return ack
```

### 6.2 Stream and consumer bootstrap

```python
# infra/nats/bootstrap.py
import asyncio
import nats
from nats.js.api import StreamConfig, ConsumerConfig, DeliverPolicy, AckPolicy

STREAMS = [
    StreamConfig(
        name="GOVERNANCE",
        subjects=["governance.>"],
        storage="file",
        retention="limits",
        max_age=60*60*24*90,   # 90 days
    ),
    StreamConfig(
        name="AGENTS",
        subjects=["agent.>"],
        storage="file",
        retention="limits",
        max_age=60*60*24*30,
    ),
    StreamConfig(
        name="DECISIONS",
        subjects=["decision.>"],
        storage="file",
        retention="limits",
        max_age=60*60*24*365,  # 1 year for audit
    ),
    StreamConfig(
        name="EXECUTION",
        subjects=["execution.>"],
        storage="file",
        retention="limits",
        max_age=60*60*24*365,
    ),
    StreamConfig(
        name="SECURITY",
        subjects=["security.>"],
        storage="file",
        retention="limits",
        max_age=60*60*24*90,
    ),
    StreamConfig(
        name="RECONCILE",
        subjects=["reconcile.>"],
        storage="file",
        retention="limits",
        max_age=60*60*24*365,
    ),
]

CONSUMERS = [
    # Firm Risk Core consumes execution plans
    ("DECISIONS", ConsumerConfig(
        name="firm-risk-core-plans",
        durable_name="firm-risk-core-plans",
        filter_subject="decision.execution_plan.created",
        ack_policy=AckPolicy.EXPLICIT,
        deliver_policy=DeliverPolicy.ALL,
        max_deliver=5,
    )),
    # Execution Router consumes approved plans
    ("EXECUTION", ConsumerConfig(
        name="execution-router-approved",
        durable_name="execution-router-approved",
        filter_subject="execution.plan.approved",
        ack_policy=AckPolicy.EXPLICIT,
        deliver_policy=DeliverPolicy.ALL,
        max_deliver=3,
    )),
    # Reconciler consumes fills
    ("EXECUTION", ConsumerConfig(
        name="reconciler-fills",
        durable_name="reconciler-fills",
        filter_subject="execution.fill.reported",
        ack_policy=AckPolicy.EXPLICIT,
        deliver_policy=DeliverPolicy.ALL,
        max_deliver=5,
    )),
    # Security controller monitors all security events
    ("SECURITY", ConsumerConfig(
        name="security-controller-all",
        durable_name="security-controller-all",
        filter_subject="security.>",
        ack_policy=AckPolicy.EXPLICIT,
        deliver_policy=DeliverPolicy.ALL,
        max_deliver=10,
    )),
]

async def bootstrap():
    nc = await nats.connect("nats://localhost:4222")
    js = nc.jetstream()
    for sc in STREAMS:
        try:
            await js.add_stream(sc)
            print(f"Stream created: {sc.name}")
        except Exception as e:
            print(f"Stream '{sc.name}' already exists or error: {e}")
    for stream_name, cc in CONSUMERS:
        try:
            await js.subscribe(
                cc.filter_subject,
                config=cc,
                stream=stream_name,
            )
            print(f"Consumer '{cc.durable_name}' bound to {stream_name}")
        except Exception as e:
            print(f"Consumer '{cc.durable_name}' error: {e}")
    await nc.close()

if __name__ == "__main__":
    asyncio.run(bootstrap())
```

---

## 7. Promotion pipeline: Research → Department Agent → Execution Planner

The promotion gate is governance-event-driven. No promotion happens through code alone.

### 7.1 Promotion event chain

```
1. Research agent produces validated ResearchPacket + StrategyValidationReport
   → published to: agent.output.research_packet, decision.validation_report.created
   → persisted to: decision_traces table

2. Security Controller runs ClawSec-style scan
   → checks: SOUL.md integrity, prompt_ref hash, tool_class allowlist compliance
   → publishes: security.integrity.scan_passed | security.integrity.scan_failed

3. Supervisor (Hermes) reviews report + security scan
   → publishes: governance.promotion.approved | governance.promotion.rejected
   → persisted to: department_promotions table

4. Policy Registry updates agents.yaml (agents.yaml is committed via PR, not hot-patched)
   → updated field: environment (research → paper | shadow)
   → new ToolGrant, RepoGrant issued by Policy Registry

5. NATS subject ACL updated to match new agents.yaml grants
   → planner gains publish rights to: decision.execution_plan.created
   → planner gains subscribe rights to: decision.risk_review.created

6. Department Runtime restarts with new agents.yaml version
   → build_agent() re-validates environment, grants, and schema
   → NATS publisher re-validates subject allowlist

7. Orchestrator API logs governance event
   → governance_events table: agent_id, from_env, to_env, policy_version, approver
```

### 7.2 Promotion gate config

```yaml
# config/governance-rules/promotion_gates.yaml
gates:
  - from_environment: research
    to_environment: paper
    required_artifacts:
      - ResearchPacketOutput        # validated, schema-conforming output
      - StrategyValidationReport    # sharpe, drawdown, bias checks
    required_security_checks:
      - soul_integrity              # SOUL.md hash must match committed version
      - prompt_policy_integrity     # prompt_ref hash must match policy registry
      - tool_class_compliance       # no blocked tool classes
    required_approvals:
      - role: firm_supervisor       # Hermes
    auto_approve: false

  - from_environment: paper
    to_environment: shadow
    required_artifacts:
      - ResearchPacketOutput
      - StrategyValidationReport
      - paper_trace_bundle          # 30+ cycles of complete paper DecisionTraces
    required_security_checks:
      - soul_integrity
      - prompt_policy_integrity
      - tool_class_compliance
      - nats_subject_scope_audit    # verify publish/subscribe subjects are within spec
    required_approvals:
      - role: firm_supervisor
      - role: department_risk       # dept risk sign-off
    auto_approve: false

  - from_environment: shadow
    to_environment: limited
    required_artifacts:
      - shadow_performance_report   # 30+ days shadow, full scorecard
      - wallet_authority_grant_request
    required_security_checks:
      - soul_integrity
      - prompt_policy_integrity
      - tool_class_compliance
      - nats_subject_scope_audit
      - repo_acl_audit              # no unexpected write/merge grants
    required_approvals:
      - role: firm_supervisor
      - role: cro_agent
      - role: human_operator        # always requires human for live transition
    auto_approve: false
    requires_capital_grant: true    # triggers CapitalAuthorityGrant issuance
```

### 7.3 Gate enforcement in loader

```python
# departments/runtime-host/promotion_guard.py
import yaml
from pathlib import Path

GATES = yaml.safe_load(
    Path("config/governance-rules/promotion_gates.yaml").read_text()
)["gates"]

def assert_promotion_permitted(agent_spec: dict, target_env: str, firm_env: str):
    """Called by build_agent before instantiating an agent in a new environment."""
    current_env = agent_spec["environment"]
    if current_env == target_env:
        return  # No promotion needed; normal instantiation

    gate = next(
        (g for g in GATES
         if g["from_environment"] == current_env
         and g["to_environment"] == target_env),
        None
    )
    if gate is None:
        raise PermissionError(
            f"No promotion gate defined from '{current_env}' to '{target_env}' "
            f"for agent '{agent_spec['id']}'. Promotion blocked."
        )
    if gate.get("auto_approve") is False:
        # In production, this check queries the governance_events table
        # to confirm that all required approvals have been recorded.
        raise PermissionError(
            f"Promotion from '{current_env}' to '{target_env}' for agent "
            f"'{agent_spec['id']}' requires explicit governance approval. "
            f"Check governance_events table before restarting runtime."
        )
```

---

## 8. Security enforcement checklist

At every startup of `departments/runtime-host/`, run the following before building any CrewAI or LangChain agent:

```python
# departments/runtime-host/startup_checks.py
from pathlib import Path
import hashlib, yaml

def compute_sha256(path: str) -> str:
    return hashlib.sha256(Path(path).read_bytes()).hexdigest()

def run_startup_checks(agents_config: dict, policy_registry_hashes: dict):
    for spec in agents_config["agents"]:
        agent_id = spec["id"]

        # 1. SOUL.md integrity
        soul_hash = compute_sha256(spec["soul_ref"])
        expected = policy_registry_hashes.get(f"{agent_id}.soul")
        assert soul_hash == expected, (
            f"SOUL.md hash mismatch for '{agent_id}': "
            f"computed={soul_hash} expected={expected}. "
            "Possible drift. Halting startup."
        )

        # 2. Prompt-policy integrity
        prompt_hash = compute_sha256(spec["prompt_ref"])
        expected_p = policy_registry_hashes.get(f"{agent_id}.prompt")
        assert prompt_hash == expected_p, (
            f"system_prompt.md hash mismatch for '{agent_id}': "
            f"computed={prompt_hash} expected={expected_p}."
        )

        # 3. No blocked tool classes in spec
        blocked = {"execution", "wallet", "signing", "broadcast"}
        declared_classes = {t["tool_class"] for t in spec.get("tools", [])}
        overlap = declared_classes & blocked
        assert not overlap, (
            f"Agent '{agent_id}' declares blocked tool class(es): {overlap}. "
            "Refusing to instantiate."
        )

        # 4. Environment matches firm env
        firm_env = "shadow"   # from CREWLESS_ENV
        assert spec["environment"] == firm_env, (
            f"Agent '{agent_id}' is configured for env '{spec['environment']}' "
            f"but firm_env='{firm_env}'. Not instantiating."
        )

        # 5. NATS publish subjects do not include restricted streams
        restricted_publish = {"execution.plan.approved", "wallet.", "governance.policy."}
        for subj in spec.get("nats", {}).get("publish", []):
            for r in restricted_publish:
                assert not subj.startswith(r), (
                    f"Agent '{agent_id}' declares publish to restricted subject "
                    f"'{subj}'. Refusing to instantiate."
                )
```

---

## 9. Repo layout additions

Add these files to the existing [crewless-capital](https://github.com/enuno/crewless-capital) structure:

```
crewless-capital/
  config/
    agents.yaml                               ← canonical agent registry
    governance-rules/
      promotion_gates.yaml                    ← gate definitions
  departments/
    runtime-host/
      loader.py                               ← YAML load + schema validation
      crewai_factory.py                       ← CrewAI agent + crew builder
      langchain_factory.py                    ← LangChain agent builder
      tool_registry.py                        ← tool_class allowlist enforcement
      model_router.py                         ← tier → provider resolution
      nats_publisher.py                       ← subject-allowlist-enforced publisher
      structured_runner.py                    ← structured output with retry + HOLD_FLAT
      startup_checks.py                       ← hash + env + tool integrity checks
      promotion_guard.py                      ← promotion gate enforcement
      tasks.py                                ← typed CrewAI Task definitions
  infra/
    nats/
      bootstrap.py                            ← stream + consumer bootstrap
      nats-server.conf                        ← JetStream config with per-subject authz
  packages/
    schemas/
      agents_schema.json                      ← generated from identity.proto
    model-routing/
      routing_table.yaml                      ← tier → primary/failover/tertiary
```

---

## 10. Key integration rules

These are non-negotiable for both CrewAI and LangChain integration:

- All agent objects are built from `agents.yaml`; no agent is hardcoded in application code.
- `soul_ref` and `prompt_ref` are loaded from versioned files; their hashes are checked against the Policy Registry at startup.
- `build_tools_for_agent()` is the only entry point for tool construction; it enforces the `tool_class` allowlist.
- `output_pydantic` (CrewAI) and `.with_structured_output()` (LangChain) are mandatory; no agent emits free-form text as a handoff artifact.
- NATS publish subjects are validated against `agents.yaml[].nats.publish` before every publish call.
- Promotion from one environment to another requires a committed agents.yaml change, a governance event in the `governance_events` table, and a runtime restart — not a live config reload.
- No agent in either framework may hold or receive private key material, wallet grants, or signing access.
- Every execution plan emitted by a planner agent is consumed by Firm Risk Core via NATS JetStream before any on-chain action is taken.
