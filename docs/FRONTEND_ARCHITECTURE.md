# Crewless Capital — Frontend Architecture Specification

**Version:** 1.0.0-draft  
**Last Updated:** 2026-04-15  
**Status:** Design — pre-implementation  
**Spec Reference:** [SPEC.md](../SPEC.md), [DEVELOPMENTPLAN.md](../DEVELOPMENTPLAN.md)  
**App Target:** `apps/dashboard/` (TypeScript / Next.js)

---

## Overview

The Crewless Capital dashboard (`apps/dashboard/`) is an **operator control plane**, not a generic web app. It surfaces the firm's internal state — capital allocations, risk posture, wallet grants, execution decisions, and immutable decision traces — to human operators in a clear, operationally safe interface.

The frontend enforces three non-negotiables derived from the [Constitutional Principles](../SPEC.md):

1. **Backend policy is authoritative.** The UI reflects permissions and status — it never replaces deterministic enforcement by `firm-risk-core` or `wallet-policy-service`.
2. **Research and execution surfaces are visually distinct.** Navigation, color, and layout must make it impossible to confuse analyst output with capital-affecting authority.
3. **Every action surface shows policy context and trace linkage before exposing controls.** No operator-facing buttons exist without visible audit context.

---

## Stack Decision: Refine + Next.js + shadcn/ui + Tailwind CSS

After evaluating Refine + Chakra UI versus Refine + shadcn/ui + Tailwind CSS, the recommended stack is:

| Layer | Choice | Rationale |
|---|---|---|
| App framework | [Refine v5](https://refine.dev) | Resource model, routing, auth, access control, audit log, and live providers map directly to Crewless Capital's domain model |
| Rendering target | Next.js App Router | Aligns with `SPEC.md §13.2` TypeScript/Next.js default for dashboard; SSR beneficial for trace/audit views |
| Component system | [shadcn/ui](https://ui.shadcn.com) | Components are copied into the repo source — full ownership, no opaque package abstraction; required for domain-specific control components |
| Styling | Tailwind CSS v4 | Utility-first; composable with shadcn; Horizon UI pattern adoption is straightforward |
| Visual reference | [Horizon UI shadcn boilerplate](https://github.com/horizon-ui/shadcn-nextjs-boilerplate) | Dashboard layout, sidebar, card, chart, and header patterns borrowed and normalized into internal design system — not consumed as a dependency |
| Icons | [Lucide React](https://lucide.dev) | Clean, consistent, tree-shakeable |
| Data viz | [Recharts](https://recharts.org) | React-native; composable; sufficient for metrics, sparklines, and allocation timelines |
| State / server | [TanStack Query](https://tanstack.com/query) | Used by Refine under the hood; explicit use for custom data fetching hooks |

### Why Not Refine + Chakra

Chakra UI integration is mature and would be faster for initial CRUD scaffolding. However, Crewless Capital requires proprietary operator UX components — `PolicyGateResultCard`, `WalletGrantScopeCard`, `KillSwitchBanner`, `DecisionTracePanel` — that need to be first-class source components, not theme overrides on opaque library widgets. shadcn/ui's source-ownership model is a better fit for a long-lived institutional control plane.

---

## Application Zones

The dashboard is organized as six primary zones, each with a distinct purpose and access model:

| Zone | Path Prefix | Primary Users | Purpose |
|---|---|---|---|
| **Executive** | `/executive` | CIO, CRO, COO, Treasury | Capital allocations, funding gates, firm health, treasury |
| **Departments** | `/departments` | Dept leads, operators | Strategy workbenches, hypotheses, validation, debates, promotions |
| **Risk** | `/risk` | CRO, risk ops | Exposure, caps, drawdowns, wallet grants, kill switches, incidents |
| **Execution** | `/execution` | Execution ops | Plans, fills, route health, chain adapter status |
| **Reconciliation** | `/reconciliation` | Ops, finance, audit | Position drift, fill reconciliation, ledger consistency |
| **Platform** | `/platform` | Engineering, SRE | Service health, agent runs, deployments, observability links |

Additional cross-cutting routes:

| Route | Purpose |
|---|---|
| `/decision-trace` | Immutable cycle traces across all zones |
| `/settings` | Users, roles, API clients |
| `/login` | Auth entry point |

---

## Full Route Tree

```
/
├── /login
│
├── /                              → Firm overview (KPI dashboard, active alerts, quick links)
│
├── /executive
│   ├── /                          → Executive overview: NAV, allocation summary, firm mode
│   ├── /allocations               → CapitalAllocationDecision history, current budgets
│   ├── /allocations/:cycleId      → Single allocation cycle detail + CIO rationale
│   ├── /funding-gates             → Department promotion/demotion gates and scorecard
│   ├── /funding-gates/:deptId     → Per-department scorecard + gate history
│   └── /treasury
│       ├── /                      → Reserve summary, yield deployments, idle capital
│       └── /events/:eventId       → TreasuryEvent detail
│
├── /departments
│   ├── /                          → Department index: all depts, state badges, KPIs
│   └── /:deptId
│       ├── /                      → Department overview: current state, budget, metrics
│       ├── /strategies            → Strategy version list
│       ├── /strategies/:versionId → Strategy version detail + artifact viewer
│       ├── /hypotheses            → Hypothesis list (from incubation / research)
│       ├── /hypotheses/:id        → Hypothesis detail + debate artifact + thesis
│       ├── /validation-runs       → Validation run list
│       ├── /validation-runs/:id   → Run detail: metrics, regime, bias audit, verdict
│       ├── /debates/:id           → DebateOutcome viewer (bull/bear artifacts)
│       └── /promotion-requests    → Pending and historical promotion requests
│
├── /risk
│   ├── /                          → Risk command center: exposure heatmap, cap status, alerts
│   ├── /exposures                 → Gross/net exposure by dept, chain, protocol
│   ├── /exposures/:id             → Exposure detail snapshot
│   ├── /policy-violations         → Policy reject log
│   ├── /policy-violations/:id     → Violation detail + FirmExecutionDecision artifact
│   ├── /wallet-grants             → WalletAuthorityGrant list (active, expired, revoked)
│   ├── /wallet-grants/:grantId    → Grant detail: scope, usage, session key, lifecycle
│   ├── /incidents
│   │   ├── /                      → Incident list (oracle, reconciliation, circuit breaker)
│   │   └── /:incidentId           → Incident detail: timeline, mitigations, linked traces
│   └── /kill-switch               → Kill switch status + arm/disarm controls (requires approval)
│
├── /execution
│   ├── /                          → Execution overview: pending plans, fill rate, chain health
│   ├── /plans                     → ExecutionPlan list
│   ├── /plans/:planId             → Plan detail: actions, grant ref, firm decision, verdict
│   ├── /fills                     → FillReport list with slippage, fee, and reconcile status
│   ├── /fills/:fillId             → Fill detail
│   └── /chain-health              → Per-chain RPC health, oracle freshness, adapter status
│
├── /reconciliation
│   ├── /                          → Reconciliation overview: last run, drift items, status
│   ├── /runs                      → ReconciliationRun list
│   ├── /runs/:runId               → Run detail: portfolio snapshot diff, discrepancies
│   └── /drift                     → Active drift items by department
│
├── /decision-trace
│   ├── /                          → Trace list: searchable by cycle_id, dept, date, verdict
│   └── /:traceId                  → Full DecisionTrace viewer
│       │                            (dept statuses → allocation → intents → plans → decision → fills)
│       └── /artifacts/:artifactType  → Raw protobuf/JSON artifact viewer
│
├── /platform
│   ├── /services                  → Service health: all apps, uptime, last heartbeat
│   ├── /agents                    → Agent run history, last output, error rate
│   ├── /deployments               → ArgoCD / K8s deployment state
│   └── /model-routing             → Model routing table: active tiers, substitution events
│
└── /settings
    ├── /users                     → User list, roles, last login
    ├── /roles                     → Role definitions and permission assignments
    └── /api-clients               → API client registry, key scopes (read-only display)
```

---

## Refine Resource Configuration

The Refine `resources` array is the canonical registry for all navigable entities. Each resource maps to list/show/create/edit/custom actions as appropriate for that domain object.

```typescript
// apps/dashboard/src/app/refine-config.ts

import type { ResourceProps } from '@refinedev/core';

export const resources: ResourceProps[] = [

  // ─── Executive ────────────────────────────────────────────────────────────
  {
    name: 'allocationCycles',
    list: '/executive/allocations',
    show: '/executive/allocations/:id',
    meta: { zone: 'executive', label: 'Allocations', icon: 'pie-chart' },
  },
  {
    name: 'fundingGates',
    list: '/executive/funding-gates',
    show: '/executive/funding-gates/:id',
    meta: { zone: 'executive', label: 'Funding Gates', icon: 'git-branch' },
  },
  {
    name: 'treasuryEvents',
    list: '/executive/treasury',
    show: '/executive/treasury/events/:id',
    meta: { zone: 'executive', label: 'Treasury', icon: 'vault' },
  },

  // ─── Departments ──────────────────────────────────────────────────────────
  {
    name: 'departments',
    list: '/departments',
    show: '/departments/:id',
    meta: { zone: 'departments', label: 'Departments', icon: 'layers' },
  },
  {
    name: 'strategyVersions',
    list: '/departments/:deptId/strategies',
    show: '/departments/:deptId/strategies/:id',
    meta: { zone: 'departments', label: 'Strategies', parent: 'departments' },
  },
  {
    name: 'hypotheses',
    list: '/departments/:deptId/hypotheses',
    show: '/departments/:deptId/hypotheses/:id',
    meta: { zone: 'departments', label: 'Hypotheses', parent: 'departments' },
  },
  {
    name: 'validationRuns',
    list: '/departments/:deptId/validation-runs',
    show: '/departments/:deptId/validation-runs/:id',
    meta: { zone: 'departments', label: 'Validation Runs', parent: 'departments' },
  },
  {
    name: 'debates',
    show: '/departments/:deptId/debates/:id',
    meta: { zone: 'departments', label: 'Debates', parent: 'departments' },
  },
  {
    name: 'promotionRequests',
    list: '/departments/:deptId/promotion-requests',
    meta: { zone: 'departments', label: 'Promotions', parent: 'departments' },
  },

  // ─── Risk ─────────────────────────────────────────────────────────────────
  {
    name: 'exposureSnapshots',
    list: '/risk/exposures',
    show: '/risk/exposures/:id',
    meta: { zone: 'risk', label: 'Exposures', icon: 'activity' },
  },
  {
    name: 'policyViolations',
    list: '/risk/policy-violations',
    show: '/risk/policy-violations/:id',
    meta: { zone: 'risk', label: 'Policy Violations', icon: 'shield-alert' },
  },
  {
    name: 'walletGrants',
    list: '/risk/wallet-grants',
    show: '/risk/wallet-grants/:id',
    meta: { zone: 'risk', label: 'Wallet Grants', icon: 'key' },
  },
  {
    name: 'incidents',
    list: '/risk/incidents',
    show: '/risk/incidents/:id',
    meta: { zone: 'risk', label: 'Incidents', icon: 'alert-triangle' },
  },

  // ─── Execution ────────────────────────────────────────────────────────────
  {
    name: 'executionPlans',
    list: '/execution/plans',
    show: '/execution/plans/:id',
    meta: { zone: 'execution', label: 'Execution Plans', icon: 'send' },
  },
  {
    name: 'fills',
    list: '/execution/fills',
    show: '/execution/fills/:id',
    meta: { zone: 'execution', label: 'Fills', icon: 'check-circle' },
  },

  // ─── Reconciliation ───────────────────────────────────────────────────────
  {
    name: 'reconciliationRuns',
    list: '/reconciliation/runs',
    show: '/reconciliation/runs/:id',
    meta: { zone: 'reconciliation', label: 'Reconciliation', icon: 'refresh-cw' },
  },
  {
    name: 'reconciliationDrift',
    list: '/reconciliation/drift',
    meta: { zone: 'reconciliation', label: 'Drift Items', parent: 'reconciliationRuns' },
  },

  // ─── Decision Trace ───────────────────────────────────────────────────────
  {
    name: 'decisionTraces',
    list: '/decision-trace',
    show: '/decision-trace/:id',
    meta: { zone: 'trace', label: 'Decision Trace', icon: 'database' },
  },

  // ─── Platform ─────────────────────────────────────────────────────────────
  {
    name: 'serviceHealth',
    list: '/platform/services',
    meta: { zone: 'platform', label: 'Services', icon: 'server' },
  },
  {
    name: 'agentRuns',
    list: '/platform/agents',
    meta: { zone: 'platform', label: 'Agents', icon: 'cpu' },
  },
  {
    name: 'modelRoutingTable',
    list: '/platform/model-routing',
    meta: { zone: 'platform', label: 'Model Routing', icon: 'git-fork' },
  },

  // ─── Settings ─────────────────────────────────────────────────────────────
  {
    name: 'users',
    list: '/settings/users',
    meta: { zone: 'settings', label: 'Users', icon: 'users' },
  },
  {
    name: 'roles',
    list: '/settings/roles',
    meta: { zone: 'settings', label: 'Roles', icon: 'shield' },
  },
];
```

---

## Refine Provider Architecture

The five core Refine providers map to Crewless Capital services:

```typescript
// apps/dashboard/src/app/providers.tsx (conceptual)

import { Refine } from '@refinedev/core';
import routerProvider from '@refinedev/nextjs-router';

// Maps to orchestrator-api /api/* endpoints
import { dataProvider } from '@/providers/data-provider';

// Session identity; roles sourced from internal RBAC
import { authProvider } from '@/providers/auth-provider';

// Action-level checks; queries /api/capabilities
import { accessControlProvider } from '@/providers/access-control-provider';

// WebSocket / SSE stream from orchestrator-api
// Drives live KPIs, incidents, chain health, fill ticker
import { liveProvider } from '@/providers/live-provider';

// Queries DecisionTrace store for audit log views
import { auditLogProvider } from '@/providers/audit-log-provider';

// Informational toasts only — never used for safety-critical alerts
import { notificationProvider } from '@/providers/notification-provider';
```

### Provider Responsibilities

| Provider | Backed by | Notes |
|---|---|---|
| `dataProvider` | `orchestrator-api` REST/GraphQL | Typed against protobuf JSON schemas from `packages/schemas/` |
| `authProvider` | `orchestrator-api /auth` | JWT session; role attached to token |
| `accessControlProvider` | `orchestrator-api /capabilities` | Capability checks per resource + action; UI reflects only — backend enforces |
| `liveProvider` | WebSocket / SSE on `orchestrator-api` | Drives live fill ticker, incident banner, chain health strip, kill switch state |
| `auditLogProvider` | `orchestrator-api /decision-traces` | Powers the DecisionTrace viewer and per-resource audit event streams |

---

## Permission Model

Frontend roles mirror backend policy categories. Every protected route and action is wrapped in a `<PermissionBoundary>` component.

### Operator Roles

| Role | Access |
|---|---|
| `exec` | All executive, risk, decision-trace, read-all |
| `risk_ops` | Risk zone (full), execution (read), reconciliation (read) |
| `dept_lead` | Own department (full), trace (read), risk (read) |
| `exec_ops` | Execution (full), reconciliation (read), risk (read) |
| `platform` | Platform zone (full), all zones (read) |
| `audit` | All zones (read-only), decision-trace (full), no actions |

### Permission Actions

- `view` — route and resource visibility
- `propose` — create proposals (allocations, promotion requests, grant issuance)
- `approve` — approve pending gates and HITL requests
- `revoke` — revoke grants, cancel plans
- `operate` — kill switch, halt/resume, incident mitigations
- `admin` — user/role management

### PermissionBoundary Component

```typescript
// apps/dashboard/src/components/foundation/PermissionBoundary.tsx

interface PermissionBoundaryProps {
  resource: string;
  action: 'view' | 'propose' | 'approve' | 'revoke' | 'operate' | 'admin';
  fallback?: React.ReactNode; // default: null (invisible, not error)
  children: React.ReactNode;
}
```

> ⚠️ `PermissionBoundary` controls UI visibility only. Enforcement of all `propose`, `approve`, `revoke`, and `operate` actions is the responsibility of `firm-risk-core`, `wallet-policy-service`, and `orchestrator-api` — never the frontend.

---

## Component Taxonomy

Components are organized into four tiers: **foundation**, **data-display**, **domain**, and **action**.

### 1. Foundation Components

App shell, layout primitives, and cross-cutting utilities.

```
foundation/
├── AppShell.tsx              // Root layout with sidebar + top bar
├── SidebarNav.tsx            // Zone-aware collapsible nav; role-filtered via accessControl
├── TopBar.tsx                // Firm mode badge, cycle status, user menu, live alerts strip
├── FirmModeBadge.tsx         // RESEARCH | PAPER | SHADOW | LIVE | RECOVERY pill
├── BreadcrumbTrail.tsx       // Context-aware breadcrumb driven by Refine route meta
├── PageHeader.tsx            // Page title + subtitle + optional action slot
├── FilterBar.tsx             // Composable filter row: search, dropdowns, date range
├── CommandPalette.tsx        // Global ⌘K search across resources and traces
├── ThemeToggle.tsx           // Light/dark mode toggle
├── PermissionBoundary.tsx    // Access control wrapper (see above)
├── ResourceStatusBanner.tsx  // Inline policy/health status bar for resource views
├── ZoneDivider.tsx           // Visual separator marking research vs. execution zones
└── KillSwitchBanner.tsx      // Full-width sticky alert when firm halt is active
```

#### KillSwitchBanner Contract

```typescript
interface KillSwitchBannerProps {
  isActive: boolean;
  haltedAt: string;        // ISO timestamp
  haltedBy: string;        // operator identity from FirmMeta
  cycleId: string;         // cycle that triggered halt
  onViewTrace: () => void;
}
```

When `isActive` is true, `KillSwitchBanner` renders as a full-width sticky bar above all other content. It is always visible regardless of route, scroll position, or permission level.

---

### 2. Data Display Components

Generic, reusable data presentation. No domain knowledge — composable building blocks.

```
data-display/
├── DataTable.tsx             // Refine-connected sortable/filterable table with pagination
├── ColumnVisibilityMenu.tsx  // Per-table column toggle
├── MetricCard.tsx            // Single KPI: value, label, trend sparkline, delta badge
├── SparklineCard.tsx         // Inline sparkline within a MetricCard or table cell
├── StatusBadge.tsx           // Configurable status chip with semantic color mapping
├── TimelinePanel.tsx         // Vertical event timeline with expandable items
├── DiffViewer.tsx            // Side-by-side or unified diff for artifact changes
├── JSONArtifactViewer.tsx    // Syntax-highlighted, collapsible JSON payload
├── ProtobufArtifactViewer.tsx // Rendered protobuf message with field annotations
├── PaginatedList.tsx         // Cursor-based paginated list for large datasets
├── EmptyState.tsx            // Contextual empty state with icon, message, and action
├── SkeletonState.tsx         // Layout-matched skeleton for loading states
├── CopyableId.tsx            // Truncated ID with copy-to-clipboard button
├── RelativeTime.tsx          // Human-readable relative timestamp with tooltip
└── ConfidencePill.tsx        // LOW | MEDIUM | HIGH badge for AllocationConfidence etc.
```

#### MetricCard Contract

```typescript
interface MetricCardProps {
  label: string;
  value: string | number;
  unit?: string;                   // '%', 'USD', 'bps'
  trend?: 'up' | 'down' | 'flat';
  delta?: string;                  // e.g. '+2.3%'
  sparklineData?: number[];
  status?: 'normal' | 'warning' | 'critical';
  traceId?: string;                // links to DecisionTrace if metric is trace-derived
}
```

#### StatusBadge Contract

```typescript
// Maps to DepartmentState, FirmMode, RiskVerdict, GrantLifecycle
type StatusVariant =
  | 'unfunded' | 'paper' | 'shadow' | 'limited' | 'full' | 'throttled' | 'paused' | 'sunsetting'
  | 'approve' | 'approve_modified' | 'reject' | 'escalate'
  | 'live' | 'recovery' | 'research'
  | 'active' | 'expired' | 'revoked' | 'pending'
  | 'normal' | 'warning' | 'critical' | 'info';

interface StatusBadgeProps {
  variant: StatusVariant;
  label?: string;   // override default label for the variant
  size?: 'sm' | 'md';
}
```

---

### 3. Domain Components

Crewless Capital-specific operational components. These encode domain semantics and must remain source-owned.

```
domain/
├── risk/
│   ├── ExposureHeatmap.tsx         // Dept × chain/protocol grid with concentration coloring
│   ├── RiskLimitCard.tsx           // Single risk cap: current value, limit, burn rate bar
│   ├── CircuitBreakerPanel.tsx     // Active breaker list with trigger conditions and state
│   ├── PolicyGateResultCard.tsx    // FirmExecutionDecision: checks_passed, checks_failed, verdict
│   └── IncidentSeverityBadge.tsx   // Incident type + severity chip
│
├── wallet/
│   ├── WalletGrantScopeCard.tsx    // WalletAuthorityGrant summary: dept, chain, protocols, caps
│   ├── GrantLifecycleTimeline.tsx  // Issued → used → expired/revoked event stream
│   ├── SessionKeyIndicator.tsx     // On-chain session key reference with status
│   └── WalletClassBadge.tsx        // DEPT_HOT | SETTLEMENT | YIELD | RECOVERY chip
│
├── allocation/
│   ├── AllocationDecisionCard.tsx  // CapitalAllocationDecision summary with dept budget diffs
│   ├── DepartmentBudgetBar.tsx     // Assigned vs utilized NAV bar with state badge
│   ├── AllocationRationalePanel.tsx // CIO rationale artifact with dimension scores
│   ├── FundingGateStepper.tsx      // Dept promotion path: UNFUNDED → PAPER → SHADOW → etc.
│   └── ScorecardTable.tsx          // Per-dept scoring dimensions in tabular form
│
├── execution/
│   ├── ExecutionPlanInspector.tsx  // ExecutionPlan actions list with chain/protocol/amount
│   ├── RouteLiquidityPanel.tsx     // Liquidity depth and slippage estimate for a route
│   ├── SlippageOutcomeCard.tsx     // Actual vs estimated slippage with deviation badge
│   ├── FirmDecisionPanel.tsx       // FirmExecutionDecision detail: verdict, passed/failed checks
│   └── ChainHealthStrip.tsx        // Per-chain: RPC status, oracle age, bridge status
│
├── department/
│   ├── DepartmentStateCard.tsx     // State badge + budget summary + health score
│   ├── StrategyVersionBadge.tsx    // Strategy version chip with promotion status
│   ├── ValidationRunSummary.tsx    // Run outcome: metrics, regime, verdict, funding gate recommendation
│   ├── DebateArtifactViewer.tsx    // Bull thesis + bear thesis + facilitator outcome
│   └── AnalystReportCard.tsx       // Per-analyst report summary with confidence score
│
├── trace/
│   ├── DecisionTraceViewer.tsx     // Full end-to-end trace: dept statuses → allocation → intents → fills
│   ├── DecisionTraceLink.tsx       // Inline link chip to a specific trace by cycle_id
│   ├── CycleArtifactTree.tsx       // Collapsible artifact tree by cycle stage
│   ├── HaltFlagList.tsx            // halt_flags array rendered with tooltips
│   └── ModelSubstitutionAlert.tsx  // Banner when model_substitutions detected in trace
│
└── reconciliation/
    ├── ReconciliationDriftCard.tsx  // Drift item: expected vs actual, dept, severity
    └── PortfolioSnapshotDiff.tsx    // Before/after position snapshot comparison
```

#### PolicyGateResultCard Contract

```typescript
import type { FirmExecutionDecision, RiskVerdict } from '@/types/generated/risk';

interface PolicyGateResultCardProps {
  decision: FirmExecutionDecision;
  showDetails?: boolean;     // expand checks_passed/checks_failed lists
  traceId: string;           // always link to source trace
  onViewTrace?: () => void;
}
```

This component must render even when `decision.allowed = false`. It is the primary surface for operator comprehension of why an execution was blocked.

#### WalletGrantScopeCard Contract

```typescript
import type { WalletAuthorityGrant } from '@/types/generated/wallet';

interface WalletGrantScopeCardProps {
  grant: WalletAuthorityGrant;
  usageSummary?: {
    txCount: number;
    totalUsdUsed: number;
    dailyCapRemaining: number;
  };
  onRevoke?: () => void;    // only rendered if PermissionBoundary('walletGrants','revoke') passes
}
```

---

### 4. Action Components

All operator-initiated actions. Every action component renders with visible policy context and produces a traceable proposal or event.

```
actions/
├── ProposeAllocationButton.tsx    // Triggers allocation review proposal
├── RequestPromotionButton.tsx     // Submits dept promotion request (HITL gate)
├── PauseStrategyButton.tsx        // Pauses dept strategy (requires confirm)
├── RevokeGrantButton.tsx          // Revokes WalletAuthorityGrant (requires confirm + reason)
├── TriggerReconcileButton.tsx     // Manually triggers reconciliation run for a dept
├── ApproveGateDialog.tsx          // HITL approval dialog: shows full scorecard before approve
├── RejectWithReasonDialog.tsx     // Rejection dialog with mandatory reason field
├── EscalateIncidentDialog.tsx     // Escalate incident: severity + assignee + note
├── HaltFirmDialog.tsx             // Kill switch confirmation: requires typed phrase + reason
├── ResumeFirmDialog.tsx           // Resume from halt: shows halt context, requires approval
└── ExportTraceButton.tsx          // Export DecisionTrace as JSON or CSV
```

#### Action Component Rules

1. Every action dialog must display relevant policy context before showing the confirm button (e.g., `ApproveGateDialog` shows the full scorecard; `HaltFirmDialog` shows current positions and NAV).
2. Every destructive or capital-affecting action requires a confirmation step — never a single click.
3. `HaltFirmDialog` requires the operator to type `HALT FIRM` before the confirm button becomes active.
4. All action payloads are typed against protobuf JSON schemas from `packages/schemas/`.
5. Action outcomes are reflected in the DecisionTrace — every action component optionally accepts a `onTraceCreated` callback to navigate to the resulting trace.

---

## Layout Shells

Three primary layout shells support different operational contexts:

### Control Shell
**Used by:** Executive, Risk, Reconciliation, Platform  
**Character:** Dense information hierarchy; left sidebar; sticky top bar with firm mode and alerts; right-side context panel for selected record details.

```
┌─────────────────────────────────────────────────────────────┐
│  TopBar: FirmModeBadge | CycleStatus | Alerts | User        │
├──────────┬──────────────────────────────────┬───────────────┤
│          │  PageHeader                       │               │
│ Sidebar  │  FilterBar                        │  Context      │
│  (zone   │  DataTable / MetricCards          │  Panel        │
│  nav)    │  ...                              │  (optional)   │
│          │                                   │               │
└──────────┴──────────────────────────────────┴───────────────┘
```

### Workbench Shell
**Used by:** Department strategy, hypothesis, debate, validation views  
**Character:** Wider canvas; collapsible side panels; tabbed artifact navigation; visual separation from execution surfaces.

```
┌─────────────────────────────────────────────────────────────┐
│  TopBar: FirmModeBadge | DeptContext | RESEARCH ZONE MARKER │
├──────────┬─────────────────────────────────────────────────┤
│          │  [Strategies] [Hypotheses] [Debates] [Validation]│
│ Sidebar  │  ─────────────────────────────────────────────  │
│          │  Artifact Content (tabs)                        │
│          │                                                  │
│          ├─────────────────────────┬────────────────────────┤
│          │  Timeline / Trace refs  │  Context artifacts     │
└──────────┴─────────────────────────┴────────────────────────┘
```

### Trace Shell
**Used by:** DecisionTrace viewer, incident detail, execution plan detail  
**Character:** Timeline-first; chronological event stream; expandable protobuf/JSON payloads; actor chips; immutability indicated by read-only chrome.

```
┌─────────────────────────────────────────────────────────────┐
│  TopBar: FirmModeBadge | CycleId | Immutable Trace Marker   │
├──────────┬──────────────────────────────────────────────────┤
│          │  CycleArtifactTree                               │
│ Sidebar  │  ── Dept Statuses                                │
│ (trace   │  ── Allocation Decision                          │
│  steps)  │  ── Dept Intents                                 │
│          │  ── Firm Decision  ← PolicyGateResultCard        │
│          │  ── Execution Plans                              │
│          │  ── Fills                                        │
│          │  ── Halt Flags / Model Substitutions             │
└──────────┴──────────────────────────────────────────────────┘
```

---

## Risk Command Center — V1 Slice Spec

The first implementation target is the **Risk Command Center** (`/risk/*`). This slice validates the full Refine + shadcn/Tailwind stack on a domain-critical surface before expanding to other zones.

### Scope

| Resource | Routes | Components |
|---|---|---|
| `exposureSnapshots` | `/risk`, `/risk/exposures`, `/risk/exposures/:id` | `ExposureHeatmap`, `RiskLimitCard`, `MetricCard`, `DataTable` |
| `policyViolations` | `/risk/policy-violations`, `/risk/policy-violations/:id` | `PolicyGateResultCard`, `DataTable`, `DecisionTraceLink` |
| `walletGrants` | `/risk/wallet-grants`, `/risk/wallet-grants/:id` | `WalletGrantScopeCard`, `GrantLifecycleTimeline`, `SessionKeyIndicator`, `RevokeGrantButton` |
| `incidents` | `/risk/incidents`, `/risk/incidents/:id` | `IncidentSeverityBadge`, `TimelinePanel`, `EscalateIncidentDialog`, `DecisionTraceLink` |
| Kill switch | `/risk/kill-switch` | `KillSwitchBanner`, `HaltFirmDialog`, `ResumeFirmDialog` |

### V1 Risk Overview Dashboard Layout

```
/risk
  ┌──────────────────────────────────────────────────────────────┐
  │ [KillSwitchBanner — only if halt active]                     │
  ├───────────┬───────────┬───────────┬───────────┬─────────────┤
  │ Firm NAV  │ Gross Exp │ Net Exp   │ Open      │ Grant       │
  │ Drawdown  │ vs Cap    │ vs Cap    │ Incidents │ Utilization │
  │ MetricCard│ MetricCard│ MetricCard│ MetricCard│ MetricCard  │
  ├─────────────────────────────┬────────────────────────────────┤
  │ ExposureHeatmap             │ Active Circuit Breakers        │
  │ (dept × chain/protocol)     │ CircuitBreakerPanel            │
  │                             ├────────────────────────────────┤
  │                             │ Recent Policy Violations       │
  │                             │ DataTable (policyViolations)   │
  ├─────────────────────────────┴────────────────────────────────┤
  │ Active Wallet Grants                                         │
  │ DataTable (walletGrants: active, expiring soon, near-cap)   │
  └──────────────────────────────────────────────────────────────┘
```

---

## TypeScript Folder Structure

```
apps/dashboard/
├── package.json
├── tsconfig.json
├── next.config.ts
├── tailwind.config.ts
├── components.json                    ← shadcn registry config
│
├── src/
│   ├── app/                           ← Next.js App Router
│   │   ├── layout.tsx                 ← Root layout: Refine, providers, fonts, global CSS
│   │   ├── page.tsx                   ← Firm overview dashboard
│   │   ├── login/
│   │   │   └── page.tsx
│   │   │
│   │   ├── executive/
│   │   │   ├── page.tsx               ← Executive overview
│   │   │   ├── allocations/
│   │   │   │   ├── page.tsx           ← AllocationCycles list
│   │   │   │   └── [id]/page.tsx      ← Single cycle detail
│   │   │   ├── funding-gates/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   └── treasury/
│   │   │       ├── page.tsx
│   │   │       └── events/[id]/page.tsx
│   │   │
│   │   ├── departments/
│   │   │   ├── page.tsx
│   │   │   └── [deptId]/
│   │   │       ├── page.tsx
│   │   │       ├── strategies/
│   │   │       │   ├── page.tsx
│   │   │       │   └── [versionId]/page.tsx
│   │   │       ├── hypotheses/
│   │   │       │   ├── page.tsx
│   │   │       │   └── [id]/page.tsx
│   │   │       ├── validation-runs/
│   │   │       │   ├── page.tsx
│   │   │       │   └── [id]/page.tsx
│   │   │       ├── debates/[id]/page.tsx
│   │   │       └── promotion-requests/page.tsx
│   │   │
│   │   ├── risk/
│   │   │   ├── page.tsx               ← Risk Command Center overview (V1 slice)
│   │   │   ├── exposures/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── policy-violations/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── wallet-grants/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [grantId]/page.tsx
│   │   │   ├── incidents/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [incidentId]/page.tsx
│   │   │   └── kill-switch/page.tsx
│   │   │
│   │   ├── execution/
│   │   │   ├── page.tsx
│   │   │   ├── plans/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [planId]/page.tsx
│   │   │   ├── fills/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [fillId]/page.tsx
│   │   │   └── chain-health/page.tsx
│   │   │
│   │   ├── reconciliation/
│   │   │   ├── page.tsx
│   │   │   ├── runs/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [runId]/page.tsx
│   │   │   └── drift/page.tsx
│   │   │
│   │   ├── decision-trace/
│   │   │   ├── page.tsx
│   │   │   └── [traceId]/
│   │   │       ├── page.tsx
│   │   │       └── artifacts/[artifactType]/page.tsx
│   │   │
│   │   ├── platform/
│   │   │   ├── services/page.tsx
│   │   │   ├── agents/page.tsx
│   │   │   ├── deployments/page.tsx
│   │   │   └── model-routing/page.tsx
│   │   │
│   │   └── settings/
│   │       ├── users/page.tsx
│   │       ├── roles/page.tsx
│   │       └── api-clients/page.tsx
│   │
│   ├── components/
│   │   ├── foundation/              ← AppShell, SidebarNav, TopBar, PermissionBoundary, etc.
│   │   ├── data-display/            ← DataTable, MetricCard, TimelinePanel, etc.
│   │   ├── domain/
│   │   │   ├── risk/
│   │   │   ├── wallet/
│   │   │   ├── allocation/
│   │   │   ├── execution/
│   │   │   ├── department/
│   │   │   ├── trace/
│   │   │   └── reconciliation/
│   │   └── actions/                 ← All action dialogs and buttons
│   │
│   ├── providers/
│   │   ├── data-provider.ts         ← REST/GraphQL adapter to orchestrator-api
│   │   ├── auth-provider.ts         ← JWT session + role resolution
│   │   ├── access-control-provider.ts
│   │   ├── live-provider.ts         ← WebSocket/SSE live data
│   │   ├── audit-log-provider.ts    ← DecisionTrace query layer
│   │   └── notification-provider.ts
│   │
│   ├── hooks/
│   │   ├── use-firm-status.ts       ← Live firm mode, kill switch state, cycle status
│   │   ├── use-exposure-summary.ts  ← Aggregated exposure for heatmap and KPIs
│   │   ├── use-active-grants.ts     ← Active wallet grants with utilization
│   │   ├── use-decision-trace.ts    ← Trace loading with artifact expansion
│   │   └── use-kill-switch.ts       ← Kill switch state + arm/disarm mutations
│   │
│   ├── types/
│   │   ├── generated/               ← TypeScript types generated from packages/schemas/
│   │   │   ├── common.ts
│   │   │   ├── department.ts
│   │   │   ├── allocation.ts
│   │   │   ├── risk.ts
│   │   │   ├── wallet.ts
│   │   │   ├── execution.ts
│   │   │   ├── treasury.ts
│   │   │   └── governance.ts
│   │   └── ui/                      ← Frontend-only types (component props, view models)
│   │       ├── permissions.ts
│   │       ├── navigation.ts
│   │       └── status-variants.ts
│   │
│   ├── lib/
│   │   ├── api-client.ts            ← Typed fetch wrapper for orchestrator-api
│   │   ├── schema-validator.ts      ← Runtime validation against packages/schemas JSON schemas
│   │   ├── format.ts                ← Currency, bps, %, timestamp formatters
│   │   ├── status-maps.ts           ← DepartmentState/FirmMode/RiskVerdict → badge variant
│   │   └── trace-helpers.ts         ← DecisionTrace navigation and artifact resolution helpers
│   │
│   ├── styles/
│   │   ├── globals.css              ← Tailwind base + design tokens (CSS vars)
│   │   └── design-tokens.css        ← Firm color palette, spacing, typography tokens
│   │
│   └── config/
│       ├── refine-config.ts         ← Resources array (see above)
│       ├── nav-config.ts            ← Zone-to-resource navigation map
│       └── permissions-config.ts    ← Role → actions capability matrix
│
├── public/
│   └── favicon.ico
│
└── tests/
    ├── components/                  ← Component unit tests
    ├── providers/                   ← Provider integration tests
    └── e2e/                         ← Playwright E2E for critical paths
        ├── risk-command-center.spec.ts
        ├── kill-switch.spec.ts
        └── decision-trace.spec.ts
```

---

## Design System Notes

### Color Semantics

Color usage is semantically constrained in the Crewless Capital dashboard:

| Color role | Token | Usage |
|---|---|---|
| Primary action | `--color-primary` (teal) | CTAs, active nav, focus rings |
| Research zone | `--color-blue` | Visual marker for research/analysis surfaces — never on execution surfaces |
| Execution zone | `--color-gold` | Visual marker for execution and live capital surfaces |
| Warning / throttled | `--color-warning` | Drawdown approaching limit, grants near cap |
| Error / critical | `--color-error` | Policy violations, circuit breaker active |
| Success / approved | `--color-success` | Approve verdict, healthy checks |
| Neutral surface | `--color-surface` family | Default surface hierarchy |

### Zone Visual Separation

Research surfaces (Workbench shell, Department zone) use a subtle blue accent on the sidebar active indicator and page header to visually distinguish them from execution surfaces (Execution zone, Risk zone), which use a gold accent. This is non-negotiable: operators must always know whether they are looking at analysis or live capital state.

### Firm Mode Badge

The `FirmModeBadge` in the `TopBar` is always visible and always accurate. It reflects the live `FirmMode` enum from the `orchestrator-api`:

- `RESEARCH` → neutral/muted
- `PAPER` → blue
- `SHADOW` → gold/amber
- `LIVE` → green with pulse animation
- `RECOVERY` → red with pulse animation

`LIVE` and `RECOVERY` modes trigger an additional ambient border on the root app shell to make mode unmistakable at a glance.

---

## Implementation Sequence

| Step | Deliverable | Aligns with Dev Plan |
|---|---|---|
| 1 | Scaffold `apps/dashboard/` with Next.js + Refine + shadcn + Tailwind; auth + app shell | Phase 3 |
| 2 | Connect `dataProvider` and `liveProvider` to `orchestrator-api` stubs | Phase 3 |
| 3 | Build Risk Command Center V1 slice (overview + exposures + grants + violations + incidents) | Phase 3/4 |
| 4 | Build Decision Trace viewer (Trace shell + `DecisionTraceViewer` + `CycleArtifactTree`) | Phase 5 |
| 5 | Build Allocation + Funding Gates views (Executive zone) | Phase 3/6 |
| 6 | Build Department Workbench views (Workbench shell + debate + validation + promotion) | Phase 5/6 |
| 7 | Build Execution zone (plans + fills + chain health) | Phase 5 |
| 8 | Build Reconciliation zone | Phase 5 |
| 9 | Build HITL approval UI (ApproveGateDialog + ResumeFirmDialog) | Phase 6 |
| 10 | Build Platform zone (service health + agent runs + model routing) | Phase 7 |

---

## Related Documents

- [SPEC.md](../SPEC.md) — System specification; protobuf contracts; firm architecture
- [DEVELOPMENTPLAN.md](../DEVELOPMENTPLAN.md) — Phase gates and service targets
- [wallet-authority.md](wallet-authority.md) — Wallet grant lifecycle (when authored)
- [api-contracts.md](api-contracts.md) — API endpoint contracts (when authored)
- [runbooks/](runbooks/) — Operational runbooks (Phase 7+)

---

> ⚠️ **Research and development status.** This dashboard is a control plane for a system that is in pre-production research/paper trading mode. No production capital is managed by this interface at this time.

---

*Built by machines. Governed by code. Owned by no one but the keyholder.*
