---
name: profitability-calculator
description: >
  Cost and breakeven engine for the arb-only CEX-DEX bundle. Computes net profitability,
  minimum spread, and sensitivity after fees, gas, slippage, transfer, and hedge costs.
  Use after a trade idea already exists. Analysis only.
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:system_get_capabilities
---

# profitability-calculator

## Role

This skill answers one narrow question:

> After all costs, is the proposed arb framework worth doing?

It does not discover opportunities and it does not decide whether the user's setup is ready.

## Input Contract

`TradeLeg[]` is unchanged from the legacy bundle.

```yaml
TradeLeg:
  venue: string
  chain: string
  asset: string
  side: string                  # "buy" | "sell"
  size_usd: number
  order_type: string            # "market" | "limit"
  estimated_slippage_bps: number
  gas_gwei: number
  withdrawal_required: boolean
  deposit_required: boolean
  bridge_required: boolean
  bridge_chain: string
  vip_tier: string
  instrument_type: string       # "spot" | "swap" | "futures"
  funding_intervals: number
  funding_rate: number
  borrow_rate_annual: number
  hold_days: number
```

## Output Contract

```yaml
ProfitabilityResult:
  net_profit_usd: number
  gross_spread_usd: number
  total_costs_usd: number
  cost_breakdown:
    cex_trading_fee: number
    dex_trading_fee: number
    gas_cost: number
    slippage_cost: number
    withdrawal_fee: number
    deposit_fee: number
    bridge_fee: number
    funding_payment: number
    borrow_cost: number
  profit_to_cost_ratio: number
  is_profitable: boolean
  min_spread_for_breakeven_bps: number
  confidence: string
  warnings: string[]
```

## Commands

### `estimate`

```bash
profitability-calculator estimate --legs '[...]'
```

Returns `ProfitabilityResult`.

### `breakdown`

Returns the same result plus per-line formulas and data sources.

### `min-spread`

Use when the caller wants the spread threshold required for breakeven.

### `sensitivity`

Use for changing size, gas, slippage, or hold window assumptions.

## Canonical Recipes

### `inventory_with_rebalance`

Use when the user already has the needed asset on both venues or can later rebalance inventory.

```yaml
legs:
  - venue: "okx-cex"
    chain: ""
    asset: "ETH"
    side: "buy"
    size_usd: 1000
    order_type: "market"
    vip_tier: "VIP0"
  - venue: "okx-dex"
    chain: "base"
    asset: "ETH"
    side: "sell"
    size_usd: 1000
    order_type: "market"
  - venue: "okx-cex"
    chain: ""
    asset: "ETH"
    side: "sell"
    size_usd: 1000
    order_type: "market"
    withdrawal_required: true
```

Interpretation:

- leg 1 and leg 2 model the arbitrage
- leg 3 models a later rebalance cost path

### `hedge_first_with_unwind`

Use when the user cannot move inventory quickly and instead locks price risk with a perp.

```yaml
legs:
  - venue: "okx-dex"
    chain: "base"
    asset: "ETH"
    side: "buy"
    size_usd: 1000
    order_type: "market"
  - venue: "okx-cex"
    chain: ""
    asset: "ETH"
    side: "sell"
    size_usd: 1000
    order_type: "market"
    instrument_type: "swap"
    funding_intervals: 1
    funding_rate: 0.0001
  - venue: "okx-cex"
    chain: ""
    asset: "ETH"
    side: "buy"
    size_usd: 1000
    order_type: "market"
    instrument_type: "swap"
```

Interpretation:

- leg 1 models the onchain fill
- leg 2 models the temporary hedge
- leg 3 models the unwind cost once transfer or inventory normalization completes

## Guidance

- Do not assume hedge legs are free; include funding and unwind slippage.
- Do not call an inventory scenario “instant” unless both sides are already funded outside this skill.
- If the trade depends on slow withdrawal, model that with `hold_days` or explicit extra costs in the calling skill.
