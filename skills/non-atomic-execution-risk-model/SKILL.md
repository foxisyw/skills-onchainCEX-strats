---
name: non-atomic-execution-risk-model
description: >
  Models the risk that one arb leg fills while the other leg slips, partially fills,
  or stalls. Use for latency windows, worst-case PnL, and retail-readable execution risk
  on CEX-DEX arbitrage. Analysis only.
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:system_get_capabilities
---

# non-atomic-execution-risk-model

## Role

This skill answers:

> What happens if my CEX and DEX legs do not complete atomically?

## Input Contract

```yaml
ExecutionRiskInput:
  asset: string
  chain: string
  size_usd: number
  framework: string               # "inventory" | "hedge-first"
  cex_depth_ratio: number         # available depth / required size
  dex_price_impact_pct: number
  latency_window_sec: number
  transfer_latency_min: number
  hedge_enabled: boolean
```

## Output Contract

```yaml
ExecutionRiskResult:
  fill_probability: number
  partial_fill_loss_usd: number
  worst_case_pnl_usd: number
  latency_window_sec: number
  cex_depth_ratio: number
  risk_verdict: string            # "LOW" | "MEDIUM" | "HIGH" | "BLOCK"
  notes: string[]
```

## Commands

### `assess`

Use for the main evaluation path.

```bash
non-atomic-execution-risk-model assess --asset ETH --chain base --size-usd 1000 --framework hedge-first
```

Decision guidance:

- `LOW`: both legs have room, latency short, worst-case loss within limits
- `MEDIUM`: trade may still work, but user should size down or hedge tightly
- `HIGH`: partial-fill pain is meaningful; do not present as retail-friendly
- `BLOCK`: worst-case loss breaches configured limits

### `stress`

Use when the user wants size or latency sensitivity.

```bash
non-atomic-execution-risk-model stress --asset ETH --chain base --size-range 500,1000,2000 --latency-range 30,60,120
```

Output:

```yaml
StressResult:
  scenarios:
    - size_usd: number
      latency_window_sec: number
      worst_case_pnl_usd: number
      risk_verdict: string
```

## Calling Rules

- `cex-dex-arbitrage.evaluate` should call `assess` for every recommended scenario.
- Inventory paths with later rebalance should use the rebalance or transfer latency, not a fake instant fill assumption.
- Hedge-first paths should explicitly include unwind timing and not only entry latency.
