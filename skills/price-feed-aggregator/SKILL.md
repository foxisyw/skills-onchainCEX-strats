---
name: price-feed-aggregator
description: >
  Retail-oriented spread detector for the arb-only CEX-DEX bundle. Produces synchronized
  OKX CEX and OnchainOS DEX price snapshots, calculates spreads, and supplies the canonical
  PriceSnapshot used by cex-dex-arbitrage. Use for price comparison, spread monitoring,
  and historical spread context. Analysis only.
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

# price-feed-aggregator

## Role

This skill is the arb bundle's spread detector.

It does four things:

1. fetch synchronized CEX + DEX prices
2. normalize them into a single `PriceSnapshot`
3. calculate raw and effective spreads
4. provide monitoring and historical context for downstream arb evaluation

It never recommends execution by itself.

## Dependencies

- OKX MCP: required
- OnchainOS CLI: required for cross-venue output
- GoPlus: not used here

If OnchainOS is unavailable and the user asked for cross-venue spread, return a clear DEX-unavailable error instead of silently degrading.

## Shared Schema

```yaml
PriceSnapshot:
  timestamp: integer
  asset: string
  chain: string | null
  mode: string                  # "analysis" | "arb"
  cex:
    venue: string               # "okx-cex"
    last: string
    bid: string
    ask: string
    data_age_ms: integer
  dex:
    venue: string               # "okx-dex:{chain}"
    price: string
    data_age_ms: integer
  gross_spread_bps: number
  buy_venue: string
  sell_venue: string
  staleness_ok: boolean
```

`arb` mode uses a 5-second staleness gate. `analysis` mode uses 60 seconds.

## Commands

### `snapshot`

Use when the user wants synchronized prices for one or more assets.

```bash
price-feed-aggregator snapshot --assets ETH,SOL --chains base,solana --mode arb
```

Returns `PriceSnapshot[]`.

### `spread`

Use when the user wants one focused spread read.

```bash
price-feed-aggregator spread --asset ETH --chain base --size-usd 1000
```

Additional output:

```yaml
SpreadResult:
  snapshot: PriceSnapshot
  estimated_cex_slippage_bps: number
  estimated_dex_slippage_bps: number
  effective_spread_bps: number
```

### `monitor`

Use for threshold alerts.

```bash
price-feed-aggregator monitor --assets ETH --chains base,arbitrum --threshold-bps 30
```

Output per alert:

```yaml
MonitorAlert:
  asset: string
  chain: string
  spread_bps: number
  threshold_bps: number
  direction: string
  suggested_next_step: string
```

### `history`

Use for historical spread context.

```bash
price-feed-aggregator history --asset ETH --chain base --lookback-days 7 --granularity 1H
```

Output:

```yaml
HistoryResult:
  asset: string
  chain: string
  lookback_days: number
  granularity: string
  avg_spread_bps: number
  max_spread_bps: number
  min_spread_bps: number
  sparkline: string
```

## Routing

Use this skill when the user asks:

- “compare CEX and DEX price”
- “價差幾多”
- “monitor spread”
- “historical spread”

Do not use it when the user is asking:

- whether the trade is executable now
- whether inventory or hedge is required
- whether the opportunity survives costs

Those belong to `cex-dex-arbitrage`.

## Operational Notes

- Use OKX bid/ask, not mid-price, when a downstream arb evaluation depends on executable direction.
- Use OnchainOS quote data for DEX pricing whenever trade size matters.
- Return strings for raw market prices, but do all arithmetic after parsing to numbers.
- If both venues are fresh but the spread is economically tiny, still report it; profitability gating happens downstream.
