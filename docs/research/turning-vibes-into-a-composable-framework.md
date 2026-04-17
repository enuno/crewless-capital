# Turning "Vibes" into a Composable R&R Framework

**Status:** Draft v0  
**Author:** Crewless Capital Research  
**Date:** 2026-04-17  
**Origin:** Internal research synthesis from `vibes.md` (0xRicker thread analysis)  
**Related:** [SPEC Addendum — R&R Framework](../../hyperliquid-trading-firm/docs/SPEC-ADDENDUM-RR-FRAMEWORK.md)

---

## Objective

Translate the conceptual insight from 0xRicker's "one framework, many bots" thread into a
formally specified, versioned, and institutionally auditable **Risk & Reward (R&R) Framework**
for Crewless Capital. This document defines the framework as a **shared, composable component**
consumed by all departments, enforced by the SAE, and computed by the PositionSizingService.

The core thesis: **edge discovery (entry logic) and edge monetization (R&R discipline) are
separable concerns.** Entry modules are swappable per department. The R&R framework is a
firm-wide protocol — versioned, non-bypassable, and testable.

---

## Table of Contents

1. [R Currency & Units](#1-r-currency--units)
2. [Strategy R&R Profile Schema](#2-strategy-rr-profile-schema)
3. [Department R&R Profiles](#3-department-rr-profiles)
4. [SAE R-Based Rules](#4-sae-r-based-rules)
5. [PositionSizingService API Contract](#5-positionsizingservice-api-contract)
6. [Framework v1 — Simple (Human-Auditable)](#6-framework-v1--simple-human-auditable)
7. [Framework v2 — Adaptive (Regime-Aware)](#7-framework-v2--adaptive-regime-aware)
8. [Component Architecture](#8-component-architecture)
9. [Risks & Failure Modes](#9-risks--failure-modes)
10. [Validation Plan](#10-validation-plan)

---

## 1. R Currency & Units

**R** is the firm's first-class risk abstraction. Every risk measurement, budget, breach
condition, and outcome metric is expressed in R units before being translated to notional
values. This ensures cross-instrument, cross-department comparability independent of price
scale, leverage, or market volatility.

### Definitions

| Symbol | Definition | Units |
|---|---|---|
| `R` | Risk per trade — the capital amount risked on a single trade | USD (or % of dept equity) |
| `R_trade` | The fixed or computed R for a specific trade instance | USD |
| `R_mult` | Outcome multiplier: realized PnL / R_trade | dimensionless (e.g., +2R, −1R) |
| `R_budget_daily` | Maximum net R loss permitted per strategy per calendar day | R units |
| `R_budget_weekly` | Maximum net R loss permitted per strategy per rolling 7-day window | R units |
| `R_open` | Sum of R at risk across all open positions for a strategy or dept | R units |
| `R_firm_open` | Sum of R_open across all depts and strategies | R units |
| `CumulativeR` | Running sum of R_mult across all closed trades | R units |
| `DrawdownR` | Peak-to-trough decline in CumulativeR | R units |
| `ExpectancyR` | (WinRate × AvgWinR) − (LossRate × AvgLossR) | R units per trade |

### R Sizing Derivation

Position size is **always derived from R**, never specified directly by a strategy agent:

```

R_trade     = dept_equity × risk_pct_per_trade
stop_dist   = |entry_price − stop_price|
position_qty = R_trade / stop_dist

```

This derivation is enforced by the **PositionSizingService** (see §5). No agent or strategy
may bypass it.

### R Expression Conventions

- Wins reported as positive multiples: `+1.5R`, `+3.2R`
- Losses reported as negative multiples: `−1.0R`, `−0.7R` (partial loss via early exit)
- Flat exits: `0R`
- DecisionTrace logs both `R_trade` (USD) and `R_mult` (dimensionless) on every close

---

## 2. Strategy R&R Profile Schema

Every strategy registered in the system must declare a `StrategyRRProfile`. This is a
**versioned config artifact** consumed by the SAE, PositionSizingService, and Dept Risk layer.

### Protobuf Definition

```protobuf
syntax = "proto3";
package crewless.rr.v0;

message StrategyRRProfile {
  string strategy_id          = 1;  // globally unique, slug format
  string version              = 2;  // semver e.g. "0.1.0"
  string dept_id              = 3;  // owning department
  string primary_time_horizon = 4;  // "5-15m" | "1-4h" | "4h-1d" | "1d+"
  RRVariant rr_variant        = 5;  // SIMPLE | ADAPTIVE

  // Risk budgets
  double max_r_per_trade      = 6;  // max R_trade as % of dept equity
  double max_r_open           = 7;  // max simultaneous open R
  double max_r_daily_loss     = 8;  // daily R budget (negative trigger)
  double max_r_weekly_loss    = 9;  // weekly R budget
  double max_consecutive_losses = 10; // count before auto-downgrade

  // Reward targets
  double target_rr_ratio      = 11; // minimum acceptable R/R (e.g. 1.5)
  double target_avg_win_r     = 12; // expected average winner in R
  repeated double tp_ladder   = 13; // partial TP levels as R multiples

  // Promotion gates
  int32  min_trades_paper     = 14; // trades required before SHADOW
  int32  min_trades_shadow    = 15; // trades required before LIMITED

  // Allowed instruments
  repeated string instruments = 16; // "perp" | "spot" | "lp" | "basis"
  repeated string chains      = 17; // "hyperliquid" | "arbitrum" | "solana"
}

enum RRVariant {
  RR_VARIANT_UNSPECIFIED = 0;
  SIMPLE                 = 1;
  ADAPTIVE               = 2;
}
```


### JSON Representation (example)

```json
{
  "strategy_id": "momentum-intraday-btc-v0",
  "version": "0.1.0",
  "dept_id": "momentum",
  "primary_time_horizon": "5-15m",
  "rr_variant": "SIMPLE",
  "max_r_per_trade": 0.0025,
  "max_r_open": 0.01,
  "max_r_daily_loss": 6.0,
  "max_r_weekly_loss": 12.0,
  "max_consecutive_losses": 4,
  "target_rr_ratio": 2.0,
  "target_avg_win_r": 2.5,
  "tp_ladder": [1.5, 2.5, 4.0],
  "min_trades_paper": 1000,
  "min_trades_shadow": 500,
  "instruments": ["perp"],
  "chains": ["hyperliquid"]
}
```


---

## 3. Department R&R Profiles

Each department defines a canonical `StrategyRRProfile` baseline. Individual strategies inherit
and may tighten (but never loosen) these values.

### Momentum / Trend Department

| Parameter | Intraday (5–15m) | Swing (1–4h) |
| :-- | :-- | :-- |
| `max_r_per_trade` | 0.25% dept equity | 0.50% dept equity |
| `max_r_open` | 1.0% dept equity | 2.0% dept equity |
| `max_r_daily_loss` | −6R | −8R |
| `max_r_weekly_loss` | −12R | −16R |
| `target_rr_ratio` | 2.0 | 1.8 |
| `target_avg_win_r` | 2.5R | 2.0R |
| `tp_ladder` | 1.5R, 2.5R, 4.0R | 1.5R, 2.0R, 3.5R |
| `max_consecutive_losses` | 4 | 5 |

### Mean Reversion / Relative Value Department

| Parameter | 1h Swing | 4–24h Swing |
| :-- | :-- | :-- |
| `max_r_per_trade` | 0.25% dept equity | 0.50% dept equity |
| `max_r_open` | 1.5% dept equity | 2.5% dept equity |
| `max_r_daily_loss` | −5R | −6R |
| `max_r_weekly_loss` | −10R | −12R |
| `target_rr_ratio` | 1.3 | 1.5 |
| `target_avg_win_r` | 1.3R | 1.5R |
| `tp_ladder` | 1.0R, 1.3R, 1.8R | 1.2R, 1.5R, 2.5R |
| `max_consecutive_losses` | 5 | 6 |

> **Note:** Mean reversion strategies accept lower R/R ratios in exchange for higher
> win rates. The key invariant is that `ExpectancyR > 0` must hold in OOS validation.

### On-Chain Event / Flow Department

| Parameter | Event-Driven | Liquidation Flow |
| :-- | :-- | :-- |
| `max_r_per_trade` | 0.30% dept equity | 0.20% dept equity |
| `max_r_open` | 0.90% dept equity | 0.60% dept equity |
| `max_r_daily_loss` | −4R | −3R |
| `max_r_weekly_loss` | −8R | −6R |
| `target_rr_ratio` | 2.5 | 3.0 |
| `target_avg_win_r` | 3.0R | 3.5R |
| `tp_ladder` | 2.0R, 3.5R, 6.0R | 2.5R, 4.0R, 7.0R |
| `max_consecutive_losses` | 3 | 3 |

### Market Neutral / Carry / Basis Department

| Parameter | Funding Arb | Basis Trade |
| :-- | :-- | :-- |
| `max_r_per_trade` | 0.15% dept equity | 0.20% dept equity |
| `max_r_open` | 2.0% dept equity | 3.0% dept equity |
| `max_r_daily_loss` | −3R | −4R |
| `max_r_weekly_loss` | −8R | −10R |
| `target_rr_ratio` | 1.2 | 1.5 |
| `target_avg_win_r` | 1.2R | 1.5R |
| `tp_ladder` | 0.8R, 1.2R | 1.0R, 1.5R |
| `max_consecutive_losses` | 6 | 5 |


---

## 4. SAE R-Based Rules

The **Safety & Allocation Engine (SAE)** enforces R-based rules deterministically. These rules
are **not LLM-gated** — they execute as pure functions against current exposure state and the
`StrategyRRProfile`. No agent output can override them.

### Rule Schema

```typescript
interface SAERRule {
  rule_id: string;           // globally unique slug
  scope: 'FIRM' | 'DEPT' | 'STRATEGY';
  trigger: RRTrigger;
  action: RRAction;
  version: string;           // semver
  bypassable: false;         // ALWAYS false — constitutional invariant
}

interface RRTrigger {
  metric: RRMetric;          // see enum below
  operator: '<' | '<=' | '>' | '>=' | '==';
  threshold: number;         // in R units or count
  window?: '1d' | '7d' | 'session' | 'position';
}

type RRAction =
  | { type: 'BLOCK_NEW_TRADES'; scope_id: string }
  | { type: 'DOWNGRADE_MODE'; target_mode: 'SHADOW' | 'PAPER' }
  | { type: 'FORCE_FLATTEN'; scope_id: string }
  | { type: 'REDUCE_MAX_R'; factor: number }
  | { type: 'ALERT_HUMAN'; severity: 'WARN' | 'CRITICAL' };

type RRMetric =
  | 'DailyNetR'
  | 'WeeklyNetR'
  | 'OpenR'
  | 'FirmOpenR'
  | 'DrawdownR'
  | 'ConsecutiveLosses'
  | 'ExpectancyR_30trade'
  | 'MeanR_100trade';
```


### Canonical Rule Set v0

```yaml
# SAE R Rules — version: 0.1.0
# Scope: STRATEGY
rules:
  - rule_id: strat-daily-r-breach
    scope: STRATEGY
    trigger: { metric: DailyNetR, operator: "<=", threshold: -max_r_daily_loss, window: 1d }
    action: { type: BLOCK_NEW_TRADES }

  - rule_id: strat-weekly-r-breach
    scope: STRATEGY
    trigger: { metric: WeeklyNetR, operator: "<=", threshold: -max_r_weekly_loss, window: 7d }
    action: { type: DOWNGRADE_MODE, target_mode: PAPER }

  - rule_id: strat-consec-loss
    scope: STRATEGY
    trigger: { metric: ConsecutiveLosses, operator: ">=", threshold: max_consecutive_losses }
    action: { type: REDUCE_MAX_R, factor: 0.5 }

  - rule_id: strat-open-r-cap
    scope: STRATEGY
    trigger: { metric: OpenR, operator: ">=", threshold: max_r_open }
    action: { type: BLOCK_NEW_TRADES }

  - rule_id: strat-expectancy-drift
    scope: STRATEGY
    trigger: { metric: ExpectancyR_30trade, operator: "<=", threshold: 0.0 }
    action: { type: DOWNGRADE_MODE, target_mode: SHADOW }

# Scope: DEPT
  - rule_id: dept-open-r-cap
    scope: DEPT
    trigger: { metric: OpenR, operator: ">=", threshold: dept_max_r_open }
    action: { type: BLOCK_NEW_TRADES, scope_id: dept_id }

  - rule_id: dept-daily-r-breach
    scope: DEPT
    trigger: { metric: DailyNetR, operator: "<=", threshold: -dept_daily_r_limit, window: 1d }
    action: { type: DOWNGRADE_MODE, target_mode: PAPER }

# Scope: FIRM
  - rule_id: firm-open-r-cap
    scope: FIRM
    trigger: { metric: FirmOpenR, operator: ">=", threshold: firm_max_open_r }
    action: { type: BLOCK_NEW_TRADES, scope_id: ALL }

  - rule_id: firm-drawdown-r-halt
    scope: FIRM
    trigger: { metric: DrawdownR, operator: ">=", threshold: firm_hard_drawdown_r }
    action: { type: FORCE_FLATTEN, scope_id: ALL }

  - rule_id: firm-drawdown-r-warn
    scope: FIRM
    trigger: { metric: DrawdownR, operator: ">=", threshold: firm_soft_drawdown_r }
    action: { type: ALERT_HUMAN, severity: CRITICAL }
```


### Suggested Firm-Level Defaults

| Parameter | Suggested Default | Notes |
| :-- | :-- | :-- |
| `firm_max_open_r` | 6% of firm equity in R | Sum of all dept OpenR |
| `firm_soft_drawdown_r` | −15% of firm equity | Triggers human alert |
| `firm_hard_drawdown_r` | −25% of firm equity | Force flatten all |
| `dept_max_r_open` | 2% of dept equity | Per dept |
| `dept_daily_r_limit` | 8R per dept | Absolute daily loss cap |


---

## 5. PositionSizingService API Contract

The **PositionSizingService** is the single, mandatory gateway for converting strategy intent
into position quantities. No agent may specify a position size directly.

### Input Contract

```typescript
interface SizingRequest {
  request_id: string;           // idempotency key (UUID)
  strategy_id: string;
  dept_id: string;
  instrument: string;           // e.g. "BTC-USD-PERP"
  chain: string;                // e.g. "hyperliquid"
  direction: 'LONG' | 'SHORT';
  entry_price: number;
  stop_price: number;           // REQUIRED — hard reject if absent or zero
  target_prices: number[];      // TP ladder in price terms
  conviction_score?: number;    // 0.0–1.0 (optional, used by ADAPTIVE variant)
  volatility_estimate?: number; // realized vol annualized (optional, ADAPTIVE only)
  regime?: string;              // "trending" | "ranging" | "volatile" (ADAPTIVE only)
  timestamp_utc: string;        // ISO8601
}
```


### Output Contract

```typescript
interface SizingResponse {
  request_id: string;           // echoed from request
  approved: boolean;
  rejection_reason?: string;    // set if approved=false
  position_qty: number;         // 0 if rejected
  r_trade_usd: number;          // R_trade in USD
  r_trade_pct: number;          // R_trade as % of dept equity
  stop_distance: number;        // |entry - stop| in price units
  effective_rr: number;         // distance to first TP / stop_distance
  max_allowed_qty: number;      // hard cap from SAE/wallet policy
  sizing_variant: 'SIMPLE' | 'ADAPTIVE';
  applied_profile_version: string;
  trace_id: string;             // links to DecisionTrace entry
}
```


### Hard Rejection Conditions

The service must return `approved: false` for any of the following — these are not soft
warnings, they are deterministic hard rejects:

1. `stop_price` is absent, zero, or equal to `entry_price`
2. `stop_price` is on the wrong side of `entry_price` for the declared direction
3. Resulting `position_qty` would cause `strategy.OpenR` to breach `max_r_open`
4. Resulting `r_trade_usd` exceeds `max_r_per_trade` cap for the strategy
5. Strategy is in `DailyNetR` breach state (blocked by SAE rule `strat-daily-r-breach`)
6. Requested instrument or chain not in strategy's `allowed_instruments` / `allowed_chains`
7. `conviction_score` present but outside `[0.0, 1.0]`

### Immutability Invariant

Once a stop is registered in a `SizingResponse`, the **stop level may not be moved adversely**
(i.e., away from entry to increase risk). The ExecutionRouter must enforce this. Stop tightening
(reducing risk) is permitted.

---

## 6. Framework v1 — Simple (Human-Auditable)

This variant is **the baseline**. It must be understandable, auditable, and verifiable by a
human in under 5 minutes. All parameters are fixed config values — no runtime inference.

### Design Principles

- Fixed `R_trade` as a constant percentage of department equity
- No runtime scaling, no volatility adjustment, no regime detection
- All parameters in a single YAML config file per strategy
- A human operator can reproduce any sizing decision with pen, paper, and a calculator


### Sizing Logic

```python
def compute_size_simple(
    entry_price: float,
    stop_price: float,
    dept_equity: float,
    profile: StrategyRRProfile,
) -> SizingResponse:

    stop_dist = abs(entry_price - stop_price)
    if stop_dist == 0:
        return SizingResponse(approved=False, rejection_reason="ZERO_STOP_DISTANCE")

    r_trade_usd = dept_equity * profile.max_r_per_trade   # fixed, no scaling
    position_qty = r_trade_usd / stop_dist

    return SizingResponse(
        approved=True,
        position_qty=position_qty,
        r_trade_usd=r_trade_usd,
        r_trade_pct=profile.max_r_per_trade,
        sizing_variant="SIMPLE",
    )
```


### Example Config (`strategy-rr-simple.yaml`)

```yaml
# Crewless Capital — Simple R&R Profile
# Human-auditable: all values are fixed constants
# Version: 0.1.0

strategy_id:        "momentum-intraday-btc-v0"
rr_variant:         SIMPLE
dept_id:            momentum

# Risk per trade — FIXED, no runtime scaling
risk_pct_per_trade: 0.0025        # 0.25% of dept equity per trade

# Open risk cap
max_r_open:         4             # max 4 simultaneous R units open

# Daily/weekly loss limits (in R units)
max_r_daily_loss:   6             # stop trading this strategy after -6R in a day
max_r_weekly_loss:  12            # downgrade to PAPER after -12R in a week

# Reward targets
target_rr_ratio:    2.0           # minimum 2:1 required to take a trade
tp_ladder_r:        [1.5, 2.5, 4.0]  # partial TPs at these R multiples

# Trade quality gate
max_consecutive_losses: 4         # halve R after 4 consecutive losses
min_rr_to_enter:    1.5           # reject signal if projected R/R < 1.5

# Promotion requirements
min_trades_paper:   1000
min_trades_shadow:  500
```


### Audit Checklist (Human-Verifiable)

Given any closed trade, a human must be able to verify all of the following in <5 minutes:

- [ ] `R_trade_usd = dept_equity × 0.0025` matches the logged value
- [ ] `position_qty = R_trade_usd / stop_distance` matches the executed size
- [ ] Stop was not moved adversely after entry
- [ ] Daily R tally at trade time was below `max_r_daily_loss`
- [ ] `DecisionTrace` entry exists with matching `trace_id`

---

## 7. Framework v2 — Adaptive (Regime-Aware)

This variant extends v1 with runtime scaling based on **regime classification**, **realized
volatility**, and **rolling strategy health metrics**. It is more capital-efficient in
favorable conditions and more defensive in adverse ones.

> **Precondition:** A strategy must graduate through Framework v1 (SIMPLE) and accumulate
> at least 500 live SHADOW trades before being promoted to ADAPTIVE. ADAPTIVE never runs
> before v1 has been validated.

### Adaptive Scaling Dimensions

| Dimension | Input Signal | Effect on R |
| :-- | :-- | :-- |
| Regime | `regime` field from RegimeMapper | Scale R up in "trending", down in "volatile" or "ranging" |
| Realized Vol | `volatility_estimate` (annualized) | Invert-scale R: high vol → smaller R |
| Strategy Health | Rolling `ExpectancyR_30trade` | Scale R down as expectancy degrades |
| Consecutive Losses | `consecutive_loss_count` | Step-down R after each threshold breach |
| Conviction Score | `conviction_score` from signal agent | Fractional scale within allowed band |

### Sizing Logic

```python
def compute_size_adaptive(
    entry_price: float,
    stop_price: float,
    dept_equity: float,
    profile: StrategyRRProfile,
    context: AdaptiveContext,
) -> SizingResponse:

    stop_dist = abs(entry_price - stop_price)
    if stop_dist == 0:
        return SizingResponse(approved=False, rejection_reason="ZERO_STOP_DISTANCE")

    # Base R from profile
    base_r_pct = profile.max_r_per_trade

    # --- Regime scalar ---
    regime_scalars = {"trending": 1.0, "ranging": 0.7, "volatile": 0.5, "unknown": 0.6}
    regime_scalar = regime_scalars.get(context.regime, 0.6)

    # --- Volatility scalar (normalize against baseline vol) ---
    baseline_vol = profile.baseline_vol  # calibrated from IS backtest
    vol_scalar = min(1.0, baseline_vol / max(context.volatility_estimate, 1e-9))
    vol_scalar = max(vol_scalar, 0.25)   # floor: never below 25% of base R

    # --- Strategy health scalar ---
    expectancy_30 = context.expectancy_r_30trade
    if expectancy_30 > 0.3:
        health_scalar = 1.0
    elif expectancy_30 > 0.0:
        health_scalar = 0.75
    else:
        health_scalar = 0.5    # negative expectancy → defensive

    # --- Conviction scalar (optional) ---
    conviction_scalar = 0.5 + 0.5 * (context.conviction_score or 0.5)  # range [0.5, 1.0]

    # --- Consecutive loss step-down ---
    loss_step_scalars = {0: 1.0, 1: 1.0, 2: 0.75, 3: 0.5, 4: 0.25}
    loss_scalar = loss_step_scalars.get(
        min(context.consecutive_losses, 4), 0.25
    )

    # --- Composite scalar, bounded ---
    composite = regime_scalar * vol_scalar * health_scalar * conviction_scalar * loss_scalar
    composite = max(0.1, min(composite, 1.0))  # hard bounds: [10%, 100%] of base R

    r_trade_pct = base_r_pct * composite
    r_trade_usd = dept_equity * r_trade_pct
    position_qty = r_trade_usd / stop_dist

    return SizingResponse(
        approved=True,
        position_qty=position_qty,
        r_trade_usd=r_trade_usd,
        r_trade_pct=r_trade_pct,
        sizing_variant="ADAPTIVE",
        applied_scalars={
            "regime": regime_scalar,
            "vol": vol_scalar,
            "health": health_scalar,
            "conviction": conviction_scalar,
            "loss_step": loss_scalar,
            "composite": composite,
        },
    )
```


### Adaptive Config Extension (`strategy-rr-adaptive.yaml`)

```yaml
# Crewless Capital — Adaptive R&R Profile Extension
# Extends simple profile; ADAPTIVE variant only
# Version: 0.1.0

strategy_id:        "momentum-intraday-btc-adaptive-v0"
rr_variant:         ADAPTIVE
extends:            "momentum-intraday-btc-v0"   # inherits all SIMPLE caps

# Regime scaling
regime_scalars:
  trending:  1.00
  ranging:   0.70
  volatile:  0.50
  unknown:   0.60

# Volatility scaling
baseline_vol:       0.65     # annualized realized vol at IS calibration
vol_scalar_floor:   0.25     # never reduce R below 25% via vol scaling

# Strategy health scaling
health_thresholds:
  strong:    0.30            # expectancy_r_30trade > 0.30 → full R
  neutral:   0.00            # 0.00–0.30 → 75% R
  weak:      -999            # < 0.0 → 50% R

# Conviction scaling
conviction_enabled: true
conviction_floor:   0.50     # min scalar from conviction signal

# Consecutive loss step-down
loss_step_scalars:
  0: 1.00
  1: 1.00
  2: 0.75
  3: 0.50
  4: 0.25

# Composite bounds
composite_floor:    0.10     # never below 10% of base R
composite_ceiling:  1.00     # never exceed base R

# Promotion gate from SIMPLE
min_trades_simple_live: 500
```


### Auditability Requirements for Adaptive

Every `SizingResponse` in ADAPTIVE mode must log `applied_scalars` to `DecisionTrace`. An
auditor must be able to reconstruct the composite scalar from the five logged inputs alone.
No "black box" inference — every scalar has a deterministic formula traceable to config.

---

## 8. Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Department Strategy Agent                                       │
│  Emits: direction, entry, stop, targets, horizon, conviction    │
└────────────────────────┬────────────────────────────────────────┘
                         │ SizingRequest (protobuf)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  PositionSizingService                                           │
│  Reads: StrategyRRProfile (versioned YAML/protobuf)             │
│  Computes: R_trade, position_qty, effective_rr                  │
│  Variant: SIMPLE | ADAPTIVE                                     │
│  Rejects: invalid stops, cap breaches                           │
└────────────────────────┬────────────────────────────────────────┘
                         │ SizingResponse (approved/rejected)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  SAE (Safety & Allocation Engine)                               │
│  Enforces: R-based rules (firm, dept, strategy scope)           │
│  Non-bypassable; deterministic rule engine                      │
│  Actions: BLOCK | DOWNGRADE | FLATTEN | ALERT                   │
└────────────────────────┬────────────────────────────────────────┘
                         │ Approved OrderIntent (protobuf)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Wallet Policy Service                                           │
│  Enforces: chain/venue/daily-notional authority grants          │
│  Converts R budgets → max notional per instrument               │
└────────────────────────┬────────────────────────────────────────┘
                         │ Signed execution authority
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Execution Router → Chain Adapter → DEX                         │
│  Enforces: stop immutability, idempotency, slippage caps        │
└────────────────────────┬────────────────────────────────────────┘
                         │ Fill events
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  DecisionTrace Store (immutable, append-only)                   │
│  Logs: R_trade, R_mult, scalars, rule triggers, fill data       │
└─────────────────────────────────────────────────────────────────┘
```


---

## 9. Risks & Failure Modes

| Risk | Description | Mitigation |
| :-- | :-- | :-- |
| Stop mis-specification | Agents declare artificially wide stops to inflate position size | PositionSizingService validates stop vs. recent ATR; reject if >N×ATR |
| Stop mutation | Agent moves stop adversely after entry | ExecutionRouter refuses stop mutations that increase risk |
| R profile staleness | Profile calibrated in a different regime | Walk-forward recalibration; regime-aware R profiles in ADAPTIVE |
| Conviction signal gaming | Agent inflates conviction to bypass vol/regime scaling | Conviction capped at 1.0; scalar bounded at [0.5, 1.0] |
| Adaptive opacity | Composite scalar too complex to audit under pressure | All scalars logged individually in DecisionTrace; scalar formula is static code |
| Cross-leg R mis-mapping | Multi-leg (basis, LP) trades hard to express as single R | Require per-leg R accounting; multi-leg R calculator per instrument type |
| Regime label lag | RegimeMapper delivers stale regime → wrong scalar applied | Timestamp regime labels; reject sizing if regime label is >N minutes old |
| Backtest overfit | R profile parameters fit to lucky IS window | Walk-forward validation; Monte Carlo on trade sequences mandatory |


---

## 10. Validation Plan

**Hypothesis:** A firm-wide R&R framework with deterministic SAE enforcement reduces
peak drawdown probability and stabilizes risk-adjusted returns across heterogeneous
strategies relative to ad-hoc per-strategy sizing.

### Data & Window

- Instruments: HyperLiquid BTC-PERP, ETH-PERP, SOL-PERP
- Horizon: 2+ years of 1-minute OHLCV + trade-level fills
- Strategies: 2–3 Momentum, 2–3 Mean Reversion, 1–2 Carry/Basis


### IS/OOS Split

- IS: oldest 65% of data
- OOS: most recent 35%
- Walk-forward: 6-month IS, 2-month OOS, rolling


### Experiments

1. **Baseline** — each strategy with ad-hoc sizing, no unified R rules
2. **v1 (SIMPLE)** — unified fixed-R sizing + SAE rule set
3. **v2 (ADAPTIVE)** — regime/vol/health-aware sizing + SAE rule set
4. **Monte Carlo** — shuffle trade sequences for all three variants (10,000 runs each)

### Acceptance Metrics

| Metric | Threshold |
| :-- | :-- |
| Max Drawdown (R) | v1/v2 < Baseline by ≥15% (median MC) |
| Sharpe (OOS) | v1/v2 ≥ Baseline |
| Ruin Probability (−30R DD) | v1/v2 < Baseline (MC) |
| Expectancy per trade | > 0.0R in OOS for each strategy |
| Days in circuit breaker | v1/v2 circuit breaker days ≤ Baseline |

### Funding Gate

| Gate | Requirement |
| :-- | :-- |
| `PAPER_ONLY` | All strategies initial state |
| `SHADOW_ALLOCATED` | v1 backtest passes + 1000 PAPER trades, no R cap breach |
| `LIMITED_CAPITAL` | 500 SHADOW trades, expectancy > 0 in OOS, no consecutive loss breach |
| `FULL_CAPITAL` | Multi-regime performance confirmed; ADAPTIVE eligible after 500 LIMITED trades |

