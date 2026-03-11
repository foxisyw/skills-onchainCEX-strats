---
name: cex-dex-arbitrage
description: >
  Retail-first flagship skill for the arb-only bundle. Checks setup readiness, scans for
  CEX-vs-DEX arbitrage, evaluates inventory and hedge-first execution frameworks, monitors
  live spread thresholds, and summarizes historical opportunity frequency. The first question
  it answers is whether the user can execute now with current setup and constraints. Analysis only.
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:market_get_candles,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:market_get_candles,
  okx-LIVE-real-money:system_get_capabilities
---

# cex-dex-arbitrage

## Role

This skill is the retail-facing orchestrator for the bundle.

Its default journey is:

1. `doctor`
2. `scan`
3. `evaluate`

It should never force a retail user to infer setup readiness from a later failure.

## Hard Rules

- Always show `[DEMO]` or `[LIVE]`.
- Always show `[RECOMMENDATION ONLY — 不會自動執行]`.
- Native tokens bypass contract-risk checks.
- Non-native tokens use GoPlus when available.
- If GoPlus is unavailable for a non-native token, label the result `UNCHECKED`.
- `UNCHECKED` opportunities may be displayed but cannot receive `READY_NOW` plus `PROCEED`.
- If OnchainOS is unavailable, `doctor` and any arb command must return `SETUP_BLOCKED`.

## Shared Vocabulary

### Readiness States

- `SETUP_BLOCKED`
- `READY_NOW`
- `NEEDS_INVENTORY`
- `NEEDS_HEDGE`
- `MARGINAL`
- `DO_NOT_TRADE`

### ExecutionProfile

`scan` and `evaluate` share one retail execution profile:

```yaml
ExecutionProfile:
  framework: string               # "auto" | "inventory" | "hedge-first"
  cex_base_ready: boolean
  cex_quote_ready: boolean
  dex_base_ready: boolean
  dex_quote_ready: boolean
  hedge_enabled: boolean
  max_transfer_latency_min: number
```

Default profile if the user says nothing:

```yaml
framework: "auto"
cex_base_ready: false
cex_quote_ready: false
dex_base_ready: false
dex_quote_ready: false
hedge_enabled: false
max_transfer_latency_min: 10
```

## Commands

### 1. `doctor`

The required first step for new users.

```bash
cex-dex-arbitrage doctor --asset ETH --chain base
```

Checks:

1. OKX MCP connectivity and mode
2. OnchainOS installation and health
3. GoPlus availability
4. risk-limit source
5. optional balance readiness when the user explicitly supplies it
6. whether hedge-first is possible in principle

Return schema:

```yaml
DoctorResult:
  setup_status: string
  blocking_issues: string[]
  warnings: string[]
  available_modes: string[]       # e.g. ["inventory", "hedge-first"]
  risk_limit_source: string       # "project defaults" | "config/risk-limits.yaml"
  recommended_next_prompt: string
```

`doctor` returns `SETUP_BLOCKED` if:

- OKX MCP is unavailable
- OnchainOS is unavailable

`doctor` returns warnings if:

- GoPlus is unavailable
- live mode was requested but the active mode is demo

### 2. `scan`

Retail discovery. Rank by readiness first, then executable profit.

```bash
cex-dex-arbitrage scan --assets ETH,SOL --chains base,arbitrum,solana --size-usd 1000
```

Parameters:

| Parameter | Default |
|-----------|---------|
| `--assets` | `BTC,ETH,SOL` |
| `--chains` | `ethereum` |
| `--size-usd` | `1000` |
| `--min-spread-bps` | project defaults |
| `--framework` | `auto` |
| `--cex-base-ready` | `false` |
| `--cex-quote-ready` | `false` |
| `--dex-base-ready` | `false` |
| `--dex-quote-ready` | `false` |
| `--hedge-enabled` | `false` |
| `--max-transfer-latency-min` | `10` |

Return schema:

```yaml
ScanResult:
  setup_status: string
  scan_params:
    assets: string[]
    chains: string[]
    size_usd: number
    execution_profile: ExecutionProfile
  opportunities:
    - asset: string
      chain: string
      readiness_status: string
      direction: string
      recommended_framework: string
      best_executable_net_profit_usd: number
      inventory_net_profit_usd: number | null
      hedge_first_net_profit_usd: number | null
      worst_case_pnl_usd: number
      mev_route: string
      trust_label: string          # "SAFE" | "WARN" | "UNCHECKED" | "BLOCK"
  blocked_assets:
    - asset: string
      chain: string
      reason: string
```

Ranking rule:

1. `READY_NOW`
2. `NEEDS_INVENTORY`
3. `NEEDS_HEDGE`
4. `MARGINAL`
5. `DO_NOT_TRADE`

Then sort by `best_executable_net_profit_usd` descending.

### 3. `evaluate`

Retail deep dive for one asset and one chain.

```bash
cex-dex-arbitrage evaluate --asset ETH --chain base --size-usd 1000 --framework auto
```

Return schema:

```yaml
EvaluateResult:
  setup_status: string
  retail_summary:
    readiness_status: string
    required_setup: string[]
    estimated_completion_time_min: string
    worst_case_loss_usd: number
    recommendation: string         # "PROCEED" | "CAUTION" | "WAIT" | "STOP"
  inventory_scenario:
    mode: string                   # "inventory_with_rebalance"
    readiness_status: string
    required_capabilities: string[]
    net_profit_usd: number | null
    total_costs_usd: number | null
    time_to_complete_min: string
    worst_case_pnl_usd: number | null
    recommended: boolean
  hedge_first_scenario:
    mode: string                   # "hedge_first_with_unwind"
    readiness_status: string
    required_capabilities: string[]
    net_profit_usd: number | null
    total_costs_usd: number | null
    time_to_complete_min: string
    worst_case_pnl_usd: number | null
    recommended: boolean
  execution_risk:
    fill_probability: number
    partial_fill_loss_usd: number
    worst_case_pnl_usd: number
    latency_window_sec: number
    risk_verdict: string
  mev_route:
    route: string
    sandwich_risk: string
    expected_success_rate: number
    expected_extra_cost_usd: number
    submission_guidance: string
```

Evaluation rule:

- Show both scenarios whenever they are logically possible.
- Never call inventory “instant” unless both sides are already funded.
- If hedge-first is not possible because `hedge_enabled=false`, show the scenario with `NEEDS_HEDGE` instead of hiding it.
- If the token is `UNCHECKED`, the top-line recommendation cannot exceed `CAUTION`.

### 4. `monitor`

Use after the user already understands setup and readiness.

```bash
cex-dex-arbitrage monitor --assets ETH --chains base --threshold-bps 50
```

Return per alert:

```yaml
MonitorAlert:
  asset: string
  chain: string
  readiness_status: string
  spread_bps: number
  recommended_framework: string
  risk_drift: string
  suggested_next_step: string
```

### 5. `backtest`

Historical context only. It does not override live readiness.

```bash
cex-dex-arbitrage backtest --asset ETH --chain base --lookback-days 30
```

Return:

```yaml
BacktestResult:
  asset: string
  chain: string
  lookback_days: number
  avg_spread_bps: number
  max_spread_bps: number
  inventory_hit_rate: number
  hedge_first_hit_rate: number
  profitable_periods: number
  sparkline: string
```

## Calling Flow

### `doctor`

1. `system_get_capabilities`
2. `which onchainos`
3. `onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
4. optional GoPlus probe for a known token
5. return `DoctorResult`

### `scan`

1. call `doctor` logic
2. for each asset/chain:
   - get `PriceSnapshot` from `price-feed-aggregator`
   - resolve direction and spread
   - apply liquidity, staleness, and price-impact gates
   - build inventory and hedge-first cost legs
   - pass legs to `profitability-calculator`
   - pass scenario assumptions to `non-atomic-execution-risk-model.assess`
   - pass DEX execution context to `mev-protected-dex-execution-plan.plan`
3. assign readiness state
4. rank output

### `evaluate`

Same as `scan`, but include both scenario blocks and the retail summary.

## Readiness Logic

Assign `SETUP_BLOCKED` when setup dependencies fail.

Assign `READY_NOW` only when:

- setup is healthy
- required framework is available under the supplied profile
- profitability meets project thresholds
- worst-case loss is within limits
- trust label is not `BLOCK` or `UNCHECKED`

Assign `NEEDS_INVENTORY` when the economics work but the user lacks funded assets on one side.

Assign `NEEDS_HEDGE` when hedge-first is the only workable framework and `hedge_enabled=false`.

Assign `MARGINAL` when profitability is positive but below the configured profit-to-cost comfort threshold.

Assign `DO_NOT_TRADE` when the trade fails profitability, trust, or risk rules.

## Example: Doctor

```text
══════════════════════════════════════════
  CEX-DEX Arbitrage Doctor
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-11 10:00 UTC
  Data sources: OKX MCP + OnchainOS
══════════════════════════════════════════

── Doctor Result ─────────────────────────

  Setup status        SETUP_BLOCKED
  OKX                 Connected (demo, read-only)
  OnchainOS           Missing
  GoPlus              Optional and unavailable
  Risk limits         project defaults
  Available modes     inventory
  Next prompt         Install OnchainOS, then rerun doctor
```

## Example: Evaluate

```text
══════════════════════════════════════════
  CEX-DEX Arbitrage Evaluate
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-11 10:05 UTC
  Data sources: OKX MCP + OnchainOS + GoPlus
══════════════════════════════════════════

── Retail Summary ────────────────────────

  Executable now?     READY_NOW
  Required setup      None
  Est. completion     2-4 min
  Worst-case loss     -$12.40
  Recommendation      PROCEED

── inventory_with_rebalance ──────────────

  Readiness           READY_NOW
  Required capability funded quote on CEX and funded base on DEX
  Net profit          +$8.10
  Total costs         $4.90
  Est. completion     2-4 min
  Worst-case PnL      -$12.40
  Recommendation      recommended

── hedge_first_with_unwind ───────────────

  Readiness           NEEDS_HEDGE
  Required capability temporary perp hedge on OKX
  Net profit          +$6.40
  Total costs         $6.60
  Est. completion     10-30 min
  Worst-case PnL      -$14.80
  Recommendation      not recommended
```

The example is intentionally consistent:

- profitable scenario clears `min_net_profit_usd`
- worst-case loss is explicit
- inventory is only `READY_NOW` because both sides are already funded
- hedge-first is not hidden just because it is not currently available
