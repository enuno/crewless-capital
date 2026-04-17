# SPEC Addendum: R&R Framework Integration

**Status:** Draft v0  
**Date:** 2026-04-17  
**Relates to:** [`SPEC.md`](../SPEC.md) — HyperLiquid Autonomous Trading Firm  
**Conceptual Origin:** [Crewless Capital — Turning "Vibes" into a Composable R&R Framework](https://github.com/enuno/crewless-capital/blob/main/docs/research/turning-vibes-into-a-composable-framework.md)  
**Framework Version:** `crewless-rr-framework@v0`

---

## Purpose

This addendum formally pins the **Crewless Capital R&R Framework v0** into the HyperLiquid
Trading Firm specification. It:

1. Establishes R as the first-class risk currency across all firm components
2. Declares `PositionSizingService` and `SAE R-rules` as **non-bypassable, versioned, testable** infrastructure
3. Extends the existing observability plan with R-metric dashboards
4. Identifies integration points with existing SPEC components (KellySizingService, RegimeMapper, SAE)

This addendum does **not** replace SPEC.md — it is additive and should be read alongside it.
Where conflicts exist, this addendum takes precedence for all R&R and sizing behavior.

---

## 1. Conceptual Origin

The framework originates from the insight that **entry logic and R&R discipline are separable
concerns** — documented in the Crewless Capital research note above. The key principle:
multiple strategies with different signals but a **shared, versioned R&R protocol** produce
more stable firm-level drawdown profiles than strategies with ad-hoc sizing.

This maps directly onto the HyperLiquid Trading Firm's multi-agent, multi-strategy
architecture: agents propose direction and stops; a shared service computes size.

---

## 2. Non-Bypassable R&R Invariants

The following behaviors are **constitutional** — no agent prompt, orchestrator instruction,
or runtime configuration may override them. Violations are treated as critical safety faults.

| # | Invariant | Enforcement Point |
|---|---|---|
| 1 | No agent may specify a position quantity directly | `PositionSizingService` hard reject |
| 2 | Every trade must declare a stop price before sizing | `PositionSizingService` hard reject |
| 3 | Stops may not be moved adversely after entry | `ExecutionRouter` mutation guard |
| 4 | R-based SAE rules fire deterministically; no LLM override | `SAE` rule engine |
| 5 | Every sizing decision is logged to `DecisionTrace` with full scalar breakdown | `DecisionTrace` append-only store |
| 6 | `StrategyRRProfile` changes require off-path versioned promotion | `ProfileRegistry` version gate |
| 7 | ADAPTIVE variant requires prior SIMPLE validation gate passage | `PromotionService` gate check |

---

## 3. Integration with Existing SPEC Components

### 3.1 KellySizingService → PositionSizingService Relationship

The existing `KellySizingService` in SPEC.md computes theoretically optimal fractional Kelly
sizes. The R&R Framework does **not** replace Kelly — it wraps it:

```

Kelly fraction (KellySizingService)
↓
min(kelly_fraction, max_r_per_trade)   ← R cap applied as hard ceiling
↓
PositionSizingService (derives qty from capped R + stop distance)

```

In practice: Kelly provides the theoretical upper bound; R profile provides the safety
ceiling; PositionSizingService computes the executable quantity. The lower of the two
always wins.

### 3.2 RegimeMapper → ADAPTIVE R Scaling

The existing `RegimeMapper` component outputs regime labels consumed by strategy agents.
In ADAPTIVE mode, these same labels are forwarded to `PositionSizingService` as the
`regime` field of `SizingRequest`. RegimeMapper labels must carry a `timestamp_utc` field;
the sizing service rejects labels older than 5 minutes.

```protobuf
// Extension to existing RegimeLabel message
message RegimeLabel {
  string regime          = 1;  // "trending" | "ranging" | "volatile" | "unknown"
  float  confidence      = 2;  // 0.0–1.0
  string timestamp_utc   = 3;  // ISO8601 — REQUIRED for adaptive sizing
  string model_version   = 4;
}
```


### 3.3 SAE Extension

Extend the existing SAE rule schema with the R-centric rule set defined in the Crewless
Capital R&R Framework document. The SAE must support these additional rule types:

```typescript
// New SAE rule actions (additive to existing SPEC.md SAE)
type NewRRActions =
  | { type: 'REDUCE_MAX_R'; factor: number; scope_id: string }
  | { type: 'DOWNGRADE_MODE'; target_mode: 'SHADOW' | 'PAPER'; scope_id: string };

// New SAE metrics to track
type NewRRMetrics =
  | 'DailyNetR'          // net R across all closed trades today
  | 'WeeklyNetR'         // rolling 7-day net R
  | 'OpenR'              // sum of R at risk in open positions
  | 'FirmOpenR'          // OpenR across all depts
  | 'DrawdownR'          // peak-to-trough CumulativeR decline
  | 'ConsecutiveLosses'  // consecutive losing trades count
  | 'ExpectancyR_30trade' // rolling 30-trade expectancy in R
```


### 3.4 WaveDetector → Conviction Signal

The existing `WaveDetector` component outputs wave confidence scores. These scores may be
forwarded as `conviction_score` in ADAPTIVE sizing requests. Scores must be normalized to
`[0.0, 1.0]` before passing to `PositionSizingService`.

---

## 4. PositionSizingService — SPEC Integration

The `PositionSizingService` described in the R&R Framework is a **new top-level service**
added to the SPEC.md service map. Place it between the strategy signal layer and the SAE.

### Service Registration

```yaml
# services/position-sizing-service
name: PositionSizingService
language: Python
inputs:
  - SizingRequest (protobuf)
  - StrategyRRProfile (from ProfileRegistry)
  - DeptEquitySnapshot (from TreasuryService)
outputs:
  - SizingResponse (protobuf)
depends_on:
  - ProfileRegistry
  - TreasuryService
  - SAEStateCache
  - DecisionTraceStore
sidecar: false
replicas: 2   # stateless; horizontal scale
health_check: /healthz
```


### ProfileRegistry

A new lightweight service responsible for storing, versioning, and serving
`StrategyRRProfile` configs:

```yaml
name: ProfileRegistry
storage: Postgres (strategy_rr_profiles table)
api:
  - GET  /profiles/{strategy_id}           # current active version
  - GET  /profiles/{strategy_id}/{version} # specific version
  - POST /profiles                         # register new profile (off-path only)
  - POST /profiles/{strategy_id}/promote   # promote version after gate check
versioning: semver
promotion_gate: requires PAPER_ONLY validation metrics to pass acceptance thresholds
```


---

## 5. R-Metric Observability Plan

### 5.1 New Prometheus Metrics

Add the following metrics to the existing Prometheus scrape targets:

```yaml
# PositionSizingService metrics
- crewless_sizing_r_trade_usd{strategy_id, dept_id, variant}      gauge
- crewless_sizing_r_trade_pct{strategy_id, dept_id, variant}      gauge
- crewless_sizing_composite_scalar{strategy_id}                   gauge   # ADAPTIVE only
- crewless_sizing_rejection_total{strategy_id, reason}            counter
- crewless_sizing_requests_total{strategy_id, approved}           counter

# SAE R-rule metrics
- crewless_sae_rule_trigger_total{rule_id, scope, action}         counter
- crewless_sae_open_r{strategy_id, dept_id}                       gauge
- crewless_sae_firm_open_r                                        gauge
- crewless_sae_daily_net_r{strategy_id}                           gauge
- crewless_sae_weekly_net_r{strategy_id}                          gauge
- crewless_sae_drawdown_r{strategy_id, dept_id}                   gauge
- crewless_sae_consecutive_losses{strategy_id}                    gauge

# Strategy R-distribution (from Reconciler / DecisionTrace)
- crewless_strategy_cumulative_r{strategy_id}                     gauge
- crewless_strategy_expectancy_r_30{strategy_id}                  gauge
- crewless_strategy_mean_r_100{strategy_id}                       gauge
- crewless_strategy_max_dd_r{strategy_id}                         gauge
```


### 5.2 Grafana Dashboard Panels

Add a new **R&R Health** dashboard with the following panel layout:

#### Row 1 — Firm Overview

| Panel | Type | Query |
| :-- | :-- | :-- |
| Firm Open R | Gauge | `crewless_sae_firm_open_r` |
| Firm Daily Net R | Stat | `sum(crewless_sae_daily_net_r)` |
| SAE Rule Triggers (24h) | Bar chart | `increase(crewless_sae_rule_trigger_total[24h])` |
| Sizing Rejections (1h) | Stat | `increase(crewless_sizing_rejection_total[1h])` |

#### Row 2 — Department R View

| Panel | Type | Query |
| :-- | :-- | :-- |
| Dept Open R (all depts) | Stacked bar | `crewless_sae_open_r` by `dept_id` |
| Dept Daily Net R | Multi-line | `crewless_sae_daily_net_r` by `dept_id` |
| Dept DrawdownR | Multi-line | `crewless_sae_drawdown_r` by `dept_id` |

#### Row 3 — Strategy R Distribution

| Panel | Type | Query |
| :-- | :-- | :-- |
| Cumulative R per Strategy | Multi-line | `crewless_strategy_cumulative_r` |
| 30-Trade Rolling Expectancy | Multi-line | `crewless_strategy_expectancy_r_30` |
| Max DrawdownR per Strategy | Bar chart | `crewless_strategy_max_dd_r` |
| Consecutive Losses | Heatmap | `crewless_sae_consecutive_losses` |

#### Row 4 — Sizing Service Internals (ADAPTIVE)

| Panel | Type | Query |
| :-- | :-- | :-- |
| Composite Scalar by Strategy | Multi-line | `crewless_sizing_composite_scalar` |
| R Trade % (actual vs max) | Multi-line | `crewless_sizing_r_trade_pct` vs profile max |
| Rejection Reasons (1h) | Pie chart | `increase(crewless_sizing_rejection_total[1h])` by `reason` |

### 5.3 Alerting Rules

```yaml
# Prometheus alerting rules (additive to existing SPEC.md alerts)
groups:
  - name: rr_framework
    rules:
      - alert: StrategyDailyRBreached
        expr: crewless_sae_daily_net_r < -5
        for: 0m
        labels: { severity: warning }
        annotations:
          summary: "Strategy {{ $labels.strategy_id }} hit daily R limit"

      - alert: FirmOpenRHigh
        expr: crewless_sae_firm_open_r > 0.05   # 5% of firm equity
        for: 2m
        labels: { severity: warning }
        annotations:
          summary: "Firm open R exceeds 5% of equity"

      - alert: FirmDrawdownRCritical
        expr: crewless_sae_drawdown_r{scope="firm"} > 0.15
        for: 0m
        labels: { severity: critical }
        annotations:
          summary: "Firm DrawdownR exceeded 15% — human review required"

      - alert: SizingRejectionSpike
        expr: increase(crewless_sizing_rejection_total[5m]) > 10
        for: 0m
        labels: { severity: warning }
        annotations:
          summary: "Unusual sizing rejection rate — potential signal or config issue"

      - alert: NegativeExpectancyDrift
        expr: crewless_strategy_expectancy_r_30 < 0
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "Strategy {{ $labels.strategy_id }} 30-trade expectancy gone negative"
```


---

## 6. Testing Requirements

All R&R framework components must have test coverage before deployment to any funded mode.

### Unit Tests

- `PositionSizingService`: all hard rejection conditions (§5 of framework doc)
- `SAE R rules`: each rule fires correctly given synthetic state snapshots
- `AdaptiveScalar`: each scaling dimension at boundary values
- `ProfileRegistry`: version promotion gate rejects unqualified profiles


### Integration Tests

- End-to-end `SizingRequest → SizingResponse → SAE check → DecisionTrace log`
- Stop mutation guard: verify ExecutionRouter rejects adverse stop moves
- SAE downgrade flow: strategy exceeds weekly R → mode transitions to PAPER
- Regime label staleness: sizing service rejects regime label >5 min old


### Paper Trading Validation Gate

Before `SHADOW_ALLOCATED`:

- Minimum 1000 trades in PAPER mode per strategy
- `ExpectancyR_30trade > 0` in final 30 trades
- Zero SAE rule breaches from misconfiguration (organic breaches acceptable)
- `DecisionTrace` completeness: 100% of trades have trace entries with full scalar logs


### Regression Suite

Pin a set of canonical sizing inputs and expected outputs as a regression test suite.
Any change to `PositionSizingService` or SAE R rules must pass this suite before merge.

```python
# Example regression fixture
REGRESSION_CASES = [
    {
        "name": "simple_btc_standard",
        "input": {
            "entry_price": 95000.0, "stop_price": 94050.0,
            "dept_equity": 100000.0, "risk_pct": 0.0025,
        },
        "expected": {
            "r_trade_usd": 250.0,
            "stop_distance": 950.0,
            "position_qty": 0.26315,   # 250 / 950
        },
    },
    {
        "name": "reject_zero_stop",
        "input": {"entry_price": 95000.0, "stop_price": 95000.0},
        "expected": {"approved": False, "rejection_reason": "ZERO_STOP_DISTANCE"},
    },
]
```


---

## 7. Open Questions

- [ ] **Multi-leg R accounting**: How do we express R for basis trades, LP positions, or
funding arb that have multiple legs with correlated risk? Requires a multi-leg R
calculator design (future addendum).
- [ ] **Cross-chain R aggregation**: FirmOpenR must aggregate across HyperLiquid, Arbitrum,
and Solana positions. Bridge latency may cause temporary double-counting. Define
reconciliation window.
- [ ] **Kelly + R interaction in ADAPTIVE**: Should Kelly fraction feed into conviction
scalar, or should they remain independent? Leaning toward independent to avoid
compounding model uncertainty.
- [ ] **R profile inheritance**: Should child strategies be able to tighten but not loosen
parent dept caps, or should they be fully independent? Current stance: tighten only.
- [ ] **Volatility estimate source**: Realized vol from HyperLiquid feed vs. implied vol
proxy? Define canonical source and update frequency for ADAPTIVE variant.

