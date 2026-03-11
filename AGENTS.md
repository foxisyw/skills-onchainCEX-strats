# Arb-Only CEX-DEX Orchestration

This repo ships one strategy family only: retail CEX-vs-DEX arbitrage.

## Skill Routing Matrix

| User Intent | Primary Skill | Supporting Skills |
|-------------|--------------|-------------------|
| “Can I use this bundle?” / “setup check” / “幫我檢查 setup” | `cex-dex-arbitrage.doctor` | - |
| “Any arb opportunities?” / “有冇得搬” | `cex-dex-arbitrage.scan` | `price-feed-aggregator`, `profitability-calculator` |
| “Evaluate this arb” / “值唔值得做” | `cex-dex-arbitrage.evaluate` | `price-feed-aggregator`, `profitability-calculator`, `non-atomic-execution-risk-model`, `mev-protected-dex-execution-plan` |
| “Monitor spread” / “監控價差” | `cex-dex-arbitrage.monitor` | `price-feed-aggregator` |
| “Historical arb frequency” / “回測” | `cex-dex-arbitrage.backtest` | `price-feed-aggregator`, `profitability-calculator` |
| “Just show me prices” / “比較價格” | `price-feed-aggregator` | - |
| “Is this profitable after costs?” / “賺唔賺” | `profitability-calculator` | - |
| “What can go wrong if fills desync?” / “非原子風險” | `non-atomic-execution-risk-model` | - |
| “Should I use private relay?” / “MEV 風險” | `mev-protected-dex-execution-plan` | - |

## Flagship Workflow

```text
1. cex-dex-arbitrage.doctor
   -> setup status + warnings + next prompt

2. cex-dex-arbitrage.scan
   -> ranked opportunities by readiness state, then executable net profit

3. cex-dex-arbitrage.evaluate
   -> inventory_with_rebalance scenario
   -> hedge_first_with_unwind scenario
   -> non-atomic-execution-risk-model.assess
   -> mev-protected-dex-execution-plan.plan / compare
   -> final retail recommendation
```

## Data Handoffs

1. Price data -> `PriceSnapshot`
2. Cost legs -> `TradeLeg[]`
3. PnL -> `ProfitabilityResult`
4. Execution profile -> `ExecutionProfile`
5. Execution risk -> `ExecutionRiskResult`
6. MEV route -> `MevRoutePlan`

## Global Rules

1. Demo by default.
2. All outputs show `[DEMO]` or `[LIVE]`.
3. All outputs show `[RECOMMENDATION ONLY — 不會自動執行]`.
4. Native tokens skip contract-risk checks; non-native tokens use GoPlus when available.
5. `UNCHECKED` non-native tokens may be shown, but cannot be recommended as `READY_NOW` + `PROCEED`.
6. Retail readiness states are mandatory:
   `SETUP_BLOCKED`, `READY_NOW`, `NEEDS_INVENTORY`, `NEEDS_HEDGE`, `MARGINAL`, `DO_NOT_TRADE`.
7. The default user journey is always `doctor -> scan -> evaluate`.
