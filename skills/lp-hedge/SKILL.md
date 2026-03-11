---
name: lp-hedge
description: >
  LP + CEX perpetual short delta-neutral yield farming strategy. Provide liquidity to high-incentive
  DeFi pools, hedge impermanent loss by shorting the volatile asset on OKX perpetual swaps.
  Only profitable with 30%+ pool APY (fees + incentives); rebalancing costs exceed LP fees otherwise.
  Trigger phrases include: "LP hedge", "LP 對沖", "delta neutral LP", "Delta 中性 LP",
  "hedge impermanent loss", "對沖無常損失", "LP farming", "LP 挖礦", "流動性挖礦",
  "liquidity mining hedge", "LP + short", "做 LP 同時對沖", "LP 收益", "LP yield",
  "impermanent loss protection", "無常損失保護", "concentrated liquidity hedge",
  "集中流動性對沖", "Uniswap hedge", "AMM hedge".
  Do NOT use for: trade execution, pure CEX basis trading (use basis-trading),
  pure funding rate arbitrage (use funding-rate-arbitrage), simple yield comparison
  without hedging (use yield-optimizer), onchain token security checks alone.
  Requires: okx-trade-mcp (CEX hedge leg), DeFiLlama MCP (pool data), GoPlus MCP (security).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:market_get_funding_rate,
  okx-DEMO-simulated-trading:market_get_instruments,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:market_get_funding_rate,
  okx-LIVE-real-money:market_get_instruments,
  okx-LIVE-real-money:system_get_capabilities
---

# lp-hedge

LP + CEX perpetual short delta-neutral yield farming skill (Skill 10) for the Onchain x CEX Strats system. Identifies high-incentive DeFi liquidity pools, calculates the precise CEX perpetual short hedge to neutralize impermanent loss exposure, and projects net yield after all costs. The strategy: **provide LP in DeFi pool + short the volatile asset on OKX perp**, earning incentive yield while remaining delta-neutral.

**Critical insight:** Without yield farming incentives >= 30% APR, rebalancing costs and hedge funding costs will exceed LP fee income. This skill MUST verify incentive APR before recommending any position.

Reuses: DeFiLlama MCP for pool discovery, OKX perpetual swap market data for hedge pricing and funding rate, GoPlus MCP for LP token contract security.

---

## 1. Role

**LP delta-neutral yield analyst** -- identifies high-incentive DeFi LP opportunities and designs CEX perpetual hedge overlays to neutralize directional exposure.

This skill is responsible for:
- Scanning DeFi pools for high-incentive LP opportunities where the volatile asset has an OKX perpetual
- Calculating the correct hedge ratio (delta) for different LP types (V2, V3 wide, V3 concentrated)
- Computing net yield after LP fees, incentives, impermanent loss, hedge funding cost, and rebalance cost
- Designing rebalancing schedules to keep the hedge aligned as price moves
- Evaluating whether a given LP + hedge combination is profitable

This skill does **NOT**:
- Execute any trades (analysis only, never sends orders or deposits liquidity)
- Handle simple yield comparisons without hedging (delegate to `yield-optimizer`)
- Handle pure CEX basis trading (delegate to `basis-trading`)
- Handle pure funding rate arbitrage (delegate to `funding-rate-arbitrage`)
- Handle CEX-DEX spot arbitrage (delegate to `cex-dex-arbitrage`)
- Manage positions, claim rewards, or rebalance on-chain

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Metric labels may use English abbreviations regardless of language:
- `delta`, `IL` (impermanent loss), `APR`, `APY`, `TVL`, `bps`, `PnL`, `funding rate`, `hedge ratio`
- `V2`, `V3`, `concentrated liquidity`, `range order`
- Timestamps always displayed in UTC

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations -- no trades, no deposits, no claims |
| Recommendation header | Always show `[RECOMMENDATION ONLY -- 不會自動執行]` |
| Live switch | Requires explicit user confirmation (see Account Safety Protocol below) |
| Leverage | **1x short ONLY** on the hedge leg. No leveraged LP or leveraged shorts. Position must be fully collateralized with separate margin. |

Even in `[LIVE]` mode, this skill only reads market data and pool data. There is no risk of accidental execution.

### Account Safety Protocol (Inlined)

Default: **demo mode** (`okx-DEMO-simulated-trading`). Switch to live requires: (1) user says "live"/"真實帳戶", (2) system shows warning and asks for confirmation, (3) user confirms, (4) `system_get_capabilities` verifies auth. If auth fails, revert to demo.

| Rule | Description |
|------|-------------|
| Default on startup | Always demo mode |
| Timeout | If no activity for 30 minutes, revert to demo mode |
| Error fallback | If live mode encounters AUTH_FAILED, revert to demo with notification |
| Header requirement | EVERY output must show `[DEMO]` or `[LIVE]` -- no exceptions |
| No auto-execution | Even in live mode, skills only provide recommendations. `[RECOMMENDATION ONLY]` header is always present. |

---

## 4. Pre-flight (Machine-Executable Checklist)

This is a **hybrid CEX + DeFi** strategy. Both okx-trade-mcp and DeFiLlama MCP are required.

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK -- output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN -- display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | DeFiLlama MCP accessible | Attempt `get_latest_pool_data` with known pool | Returns non-empty pool data | BLOCK -- output `DEFI_DATA_UNAVAILABLE`. Cannot scan LP pools without DeFiLlama. |
| 4 | GoPlus MCP accessible | Attempt `check_token_security` with known token | Returns security data object | WARN -- output `SECURITY_UNAVAILABLE`. Proceed with caution disclaimer. |
| 5 | SWAP instruments accessible | `market_get_instruments(instType: "SWAP")` | Returns non-empty array of perpetual swap contracts | BLOCK -- output `INSTRUMENT_NOT_FOUND`. Cannot hedge without perpetual swaps. |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX data for hedging)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> BLOCK (cannot discover LP pools without DeFiLlama)
    Check 3 PASS -> Check 4
      Check 4 FAIL -> WARN (security unchecked, add disclosure to all output)
      Check 4 PASS -> Check 5
        Check 5 FAIL -> BLOCK (cannot hedge without perpetual instruments)
        Check 5 PASS -> ALL SYSTEMS GO
```

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "LP 挖礦點樣對沖" / "hedge my LP" | Yes -- `evaluate` or `hedge-calc` | -- |
| "邊個 pool 可以做 delta neutral" / "hedgeable pools" | Yes -- `scan` | -- |
| "我個 LP 要唔要 rebalance" / "should I rebalance" | Yes -- `rebalance-check` | -- |
| "計個 hedge size 畀我" / "calculate hedge" | Yes -- `hedge-calc` | -- |
| "資金費率幾多" / "funding rate" | No | `funding-rate-arbitrage` |
| "基差幾多" / "futures basis" | No | `basis-trading` |
| "CEX 同 DEX 價差" / "CEX-DEX spread" | No | `cex-dex-arbitrage` |
| "純收益比較唔使對沖" / "yield comparison only" | No | `yield-optimizer` |
| "呢個幣安唔安全" / "is this token safe" | No | GoPlus MCP directly |
| "幫我開倉" / "execute trade" | No | Refuse -- no skill executes trades |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Scan hedgeable LP pools | Read | Discover high-incentive DeFi pools where the volatile asset has an OKX perp, filter by 30%+ APY and $1M+ TVL |
| `evaluate` | Deep analysis of a specific pool + hedge | Read | Full cost/yield projection for a specific LP pool with CEX short hedge overlay |
| `hedge-calc` | Calculate hedge parameters | Read | Given a pool and position size, compute exact short size, delta, and rebalance triggers |
| `rebalance-check` | Rebalancing evaluation | Read | Given current position state, determine if hedge needs rebalancing and compute adjustment |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Discover high-incentive DeFi LP pools that can be hedged with an OKX perpetual short.

```bash
lp-hedge scan --chains ethereum,arbitrum,base --min-apy-pct 30 --min-tvl-usd 1000000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--chains` | string[] | No | All supported | `ethereum`, `arbitrum`, `base`, `optimism`, `polygon`, `bsc`, `avalanche`, `solana` | Comma-separated, lowercase. |
| `--min-apy-pct` | number | No | `30` | -- | Min: 10 (hard floor). Minimum total pool APY (fees + incentives). Values below 30 trigger a WARN. |
| `--min-tvl-usd` | number | No | `1,000,000` | -- | Min: 100,000 (hard floor). Minimum pool TVL in USD. |
| `--assets` | string[] | No | All hedgeable | Any token symbols | Filter pools that contain at least one of these assets. Uppercase, comma-separated. |
| `--lp-type` | string | No | `"all"` | `v2`, `v3`, `stable`, `all` | Filter by AMM type. |
| `--top-n` | integer | No | `10` | 1 - 50 | Maximum results to return. Positive integer. |

#### Return Schema

```yaml
LPHedgeScanResult:
  timestamp: integer
  total_pools_scanned: integer
  hedgeable_pools_found: integer
  results:
    - pool_name: string            # e.g. "ETH-USDC 0.3%"
      protocol: string             # e.g. "Uniswap V3"
      chain: string
      asset_volatile: string       # Asset to hedge (e.g. "ETH")
      tvl_usd: number
      fee_apr_pct: number          # pool_fee_tier * daily_volume / tvl * 365
      incentive_apr_pct: number    # reward_tokens_per_day * token_price * 365 / tvl
      total_apr_pct: number        # fee + incentive (must be >= 30%)
      perp_instId: string          # e.g. "ETH-USDT-SWAP"
      funding_rate_annual_pct: number  # funding_rate_8h * 3 * 365 * 100
      estimated_net_yield_pct: number  # total_apr - funding - rebalance - residual_il
      risk_level: string           # "[SAFE]", "[WARN]", "[BLOCK]"
      security_status: string      # "verified" | "flagged" | "unchecked"
      warnings: string[]
```

---

### 7.2 Command: `evaluate`

Deep analysis of a specific LP pool with a full CEX hedge overlay cost/yield projection.

```bash
lp-hedge evaluate --pool "ETH-USDC" --protocol "Uniswap V3" --chain arbitrum --size-usd 50000 --lp-type v3_concentrated --range-pct 10
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--pool` | string | Yes | -- | Pool pair name | Non-empty (e.g. "ETH-USDC"). |
| `--protocol` | string | Yes | -- | Protocol name | Non-empty (e.g. "Uniswap V3", "Aerodrome"). |
| `--chain` | string | Yes | -- | Chain name | Supported chain. |
| `--size-usd` | number | No | `50,000` | -- | Min: 1,000, Max: 1,000,000 (hard cap). |
| `--lp-type` | string | No | `"v2"` | `v2`, `v3_wide`, `v3_concentrated` | AMM type determines delta calculation. |
| `--range-pct` | number | No | `10` | -- | Only for v3_concentrated: range width as +/- percentage. Min: 1, Max: 100. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | OKX fee tier for hedge leg. |
| `--hold-days` | integer | No | `90` | -- | Projected holding period in days. Min: 7, Max: 365. |

#### Return Schema

```yaml
LPHedgeEvaluateResult:
  timestamp: integer
  pool: string
  protocol: string
  chain: string
  size_usd: number
  lp_type: string
  range_pct: number                      # null for v2
  market_data:
    volatile_asset: string               # e.g. "ETH"
    volatile_price: number
    perp_instId: string                  # e.g. "ETH-USDT-SWAP"
    funding_rate_8h: number
    funding_rate_annual_pct: number
  pool_data:
    tvl_usd: number
    fee_apr_pct: number
    incentive_apr_pct: number
    incentive_token: string
    incentive_duration_days: integer
    total_apr_pct: number
  hedge_parameters:
    delta: number                        # 0.50 (V2), 0.65 (V3 ±10%), etc.
    short_amount: number                 # Units to short
    short_notional_usd: number
    margin_required_usd: number          # 1x collateral
  cost_analysis:
    total_entry_cost: number
    ongoing_costs_annual:                # funding + rebalance + residual IL
      hedge_funding_usd: number
      rebalance_cost_usd: number
      residual_il_usd: number
    total_exit_cost: number
    total_cost_for_period_usd: number
  yield_projection:
    total_income_usd: number
    net_profit_usd: number
    net_yield_annual_pct: number
    break_even_days: number
    profit_to_cost_ratio: number
  risk_assessment:
    risk_score: integer                  # 1-10
    risk_gauge: string
    risk_label: string
    warnings: string[]
  security_check:
    lp_token_status: string
    reward_token_status: string
  scenarios:
    - label: string                      # "Price +20%", "Price -20%", "Flat"
      net_pnl_usd: number
      net_yield_pct: number
  is_profitable: boolean
  recommendation: string
```

---

### 7.3 Command: `hedge-calc`

Calculate precise hedge parameters for a given LP position.

```bash
lp-hedge hedge-calc --asset ETH --size-usd 10000 --lp-type v3_concentrated --range-pct 5
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any token with OKX perp | Uppercase. |
| `--size-usd` | number | Yes | -- | -- | Min: 1,000. Total LP position size. |
| `--lp-type` | string | No | `"v2"` | `v2`, `v3_wide`, `v3_concentrated` | Determines delta formula. |
| `--range-pct` | number | No | `10` | -- | For V3: range width as +/- %. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | OKX fee tier. |

#### Return Schema

```yaml
HedgeCalcResult:
  timestamp: integer
  asset: string
  asset_price: number
  size_usd: number
  lp_type: string
  delta: number
  short_amount: number               # Units to short on OKX
  short_notional_usd: number
  margin_required_usd: number        # 1x collateral
  rebalance_triggers:
    - condition: string              # "Price > ±20%"
      timeframe: string              # "IMMEDIATELY"
      adjustment_amount: number
  cost_per_rebalance_usd: number
  estimated_rebalances_per_year: integer
  annual_rebalance_cost_usd: number
```

---

### 7.4 Command: `rebalance-check`

Evaluate whether an existing LP + hedge position needs rebalancing.

```bash
lp-hedge rebalance-check --asset ETH --entry-price 3000 --current-price 3450 --short-amount 0.833 --size-usd 10000 --lp-type v2 --hours-since-last 8
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any token | Uppercase. |
| `--entry-price` | number | Yes | -- | -- | Price when hedge was set. |
| `--current-price` | number | No | -- | -- | If omitted, fetch from `market_get_ticker`. |
| `--short-amount` | number | Yes | -- | -- | Current short position in asset units. |
| `--size-usd` | number | Yes | -- | -- | Total LP position size in USD. |
| `--lp-type` | string | No | `"v2"` | `v2`, `v3_wide`, `v3_concentrated` | AMM type. |
| `--range-pct` | number | No | `10` | -- | For V3 positions. |
| `--hours-since-last` | number | No | `0` | -- | Hours since last rebalance. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Fee tier. |

#### Return Schema

```yaml
RebalanceCheckResult:
  timestamp: integer
  asset: string
  price_change_pct: number
  current_short: number
  required_short: number
  adjustment_amount: number          # Positive = increase, negative = decrease
  direction: string                  # "INCREASE_SHORT" | "DECREASE_SHORT" | "NO_CHANGE"
  rebalance_decision: string         # "REBALANCE_NOW" | "WAIT" | "NO_CHANGE_NEEDED"
  triggered_rule: string
  rebalance_cost_usd: number
```

---

## 8. Execution Flow

### Step 1: Intent Recognition

Parse user message to extract:

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `hedge-calc` / `rebalance-check` | Default: `scan` |
| Pool / Protocol | Extract pool name (e.g. "ETH-USDC on Uniswap") | For `scan`: all. For `evaluate`: **ask user**. |
| Chain | Look for chain names | Default: all chains |
| Size | Look for "$X", "X USDT", "X 美金" | Default: $50,000 |
| LP type | Look for "V2", "V3", "concentrated" | Default: V2 |
| Range | Look for "±5%", "±10%", "range 5%" | Default: ±10% for V3 |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "邊個 pool", "hedgeable pools", "搵 pool" | `scan` |
| "評估", "evaluate", "分析", "值唔值得", "profitable" | `evaluate` |
| "計 hedge", "hedge-calc", "對沖幾多", "short 幾多", "計 delta" | `hedge-calc` |
| "要唔要 rebalance", "rebalance", "再平衡", "check hedge" | `rebalance-check` |

### Step 2: Data Collection

#### For `scan` command:

1. DeFiLlama: `get_latest_pool_data` -> filter: `total_apr >= 30%`, `tvl >= $1M`
2. OKX: `market_get_instruments(instType: "SWAP")` -> build hedgeable asset set
3. Cross-reference: keep pools where volatile asset has OKX perp
4. For each pool: `market_get_ticker` (price), `market_get_funding_rate` (funding), GoPlus checks
5. Calculate: `net_yield = total_apr - funding_cost - rebalance_cost - residual_il`
6. Filter `net_yield > 0` + security pass, sort descending, return top-n

#### For `evaluate` command:

1. DeFiLlama: pool-specific data (tvl, volume, fee_apr, incentive_apr, incentive_token, duration)
2. OKX: `market_get_ticker` (spot price), `market_get_funding_rate` (funding), `market_get_orderbook` (slippage)
3. GoPlus: `check_token_security` on LP token + reward token
4. Calculate all parameters (Section 9), run scenario analysis (+20%, -20%, flat)

#### For `hedge-calc` command:

1. OKX: `market_get_ticker` for current price
2. Look up delta by lp_type (Section 9.1)
3. Calculate: `short_amount = delta * volatile_value_in_lp / current_price`
4. Build rebalance trigger table, estimate annual rebalance cost

#### For `rebalance-check` command:

1. Fetch current price (or use `--current-price`)
2. Calculate price change %, recalculate required short
3. Apply rebalance rules (Section 12), calculate adjustment cost

**Important:** OKX returns all values as strings. Always `parseFloat()` before arithmetic.

### Step 3: Compute

(All formulas detailed in Section 9 below.)

### Step 4: Format Output

Use these output templates:

- **Header:** Global Header Template (skill icon: Shield) with mode
- **Body:** Command-specific template (see Section 13)
- **Footer:** Next Steps Template

(All templates are inlined in Section 13 below.)

---

## 9. Formulas

### 9.1 LP Delta by AMM Type

```
LP Delta = fraction of LP value exposed to directional price movement of the volatile asset.

For a two-asset pool with one volatile asset (e.g. ETH) and one stable asset (e.g. USDC):

  Uniswap V2 (full range):
    delta = 0.50
    Reason: V2 LP always holds 50/50 value split. A 1% price increase
    causes the LP to rebalance, reducing ETH and gaining USDC. The net
    exposure to ETH price movement is exactly 50% of the total position.

  Uniswap V3 wide range (±20%):
    delta = 0.55
    Reason: Wider-than-V2 but still concentrated. Slightly more directional
    exposure than V2 because liquidity is concentrated in a narrower band.

  Uniswap V3 concentrated (±10% range):
    delta = 0.65
    Reason: More concentrated liquidity means more inventory turnover per
    price movement, increasing directional exposure.

  Uniswap V3 concentrated (±5% range):
    delta = 0.75
    Reason: Very concentrated. The position behaves more like a 75/25 split.

  Uniswap V3 concentrated (±2% range):
    delta = 0.85
    Reason: Extremely concentrated. Nearly all exposure is directional.

  Stable pair (e.g. USDC-USDT):
    delta = 0.00
    Reason: No directional exposure (both assets are stable). No hedge needed.

Worked Example (V2, ETH-USDC, $10,000 LP):
  Total LP = $10,000
  ETH portion = $5,000 (50/50 split)
  ETH price = $3,000
  delta = 0.50

  short_eth_amount = delta * eth_value_in_lp / eth_price
                   = 0.50 * $5,000 / $3,000
                   = 0.833 ETH to short

Worked Example (V3 ±10%, ETH-USDC, $10,000 LP):
  Total LP = $10,000
  ETH portion = $5,000
  ETH price = $3,000
  delta = 0.65

  short_eth_amount = 0.65 * $5,000 / $3,000
                   = 1.083 ETH to short

Worked Example (V3 ±5%, ETH-USDC, $50,000 LP):
  Total LP = $50,000
  ETH portion = $25,000
  ETH price = $3,000
  delta = 0.75

  short_eth_amount = 0.75 * $25,000 / $3,000
                   = 6.25 ETH to short
  short_notional_usd = 6.25 * $3,000 = $18,750
  margin_required_usd = $18,750 (1x, fully collateralized)
```

### 9.2 Impermanent Loss Formula

```
IL = 2 * sqrt(price_ratio) / (1 + price_ratio) - 1

Variables:
  price_ratio = new_price / original_price

IL is always <= 0 (a loss relative to holding).
IL = 0 only when price_ratio = 1.0 (price unchanged).

Quick Reference Table:

  Price Change | Price Ratio | IL (unhedged) | IL (hedged, approx residual)
  ─────────────┼─────────────┼───────────────┼─────────────────────────────
  ±5%          | 0.95 / 1.05 | -0.06%        | ~0.01%
  ±10%         | 0.90 / 1.10 | -0.23%        | ~0.03%
  ±20%         | 0.80 / 1.20 | -0.94%        | ~0.10%
  ±50%         | 0.50 / 1.50 | -5.72%        | ~0.50%
  ±100%        | 0.50 / 2.00 | -25.0%        | ~2.0%

The "hedged" residual IL comes from rebalancing slippage -- the hedge cannot
perfectly track continuous LP rebalancing, so a small residual remains.

Worked Example:
  Original ETH price: $3,000
  Current ETH price:  $3,600
  price_ratio = 3600 / 3000 = 1.20

  IL = 2 * sqrt(1.20) / (1 + 1.20) - 1
     = 2 * 1.09545 / 2.20 - 1
     = 2.19089 / 2.20 - 1
     = 0.99586 - 1
     = -0.00414
     = -0.41%

  On a $50,000 LP: IL = $50,000 * -0.00414 = -$207.00 (unhedged)
  With delta-neutral hedge (residual): ~$50,000 * -0.0010 = -$50.00
```

### 9.3 Net Yield Formula

```
net_yield = lp_fees_apr + incentive_apr - residual_il_cost - hedge_funding_cost - rebalance_cost

Where:
  lp_fees_apr = pool_fee_tier * daily_volume / tvl * 365
  incentive_apr = reward_tokens_per_day * reward_token_price * 365 / tvl
  residual_il_cost = estimated 0.1% - 0.5% annual (from rebalance lag and slippage)
  hedge_funding_cost = abs(funding_rate_annual) * hedge_ratio
  rebalance_cost = (cost_per_rebalance * rebalances_per_year) / position_size

Variables:
  pool_fee_tier       = AMM fee tier (e.g. 0.003 for 0.3%)
  daily_volume        = 24-hour trading volume through the pool (USD)
  tvl                 = Total value locked in the pool (USD)
  funding_rate_annual = current_8h_rate * 3 * 365
  hedge_ratio         = delta (from Section 9.1, e.g. 0.50 for V2)
  cost_per_rebalance  = gas + slippage + trading fee (typically $50-$100)
  rebalances_per_year = estimated frequency (typically 12-24 for V2, 24-52 for V3)

Worked Example ($50,000 LP, ETH-USDC V2 on Arbitrum):

  lp_fees_apr:
    pool_fee_tier = 0.003 (0.3%)
    daily_volume = $5,000,000
    tvl = $20,000,000
    lp_fees_apr = 0.003 * 5000000 / 20000000 * 365 = 0.274 = 27.4%
    (But we only get our share: for simplicity, DeFiLlama gives this directly)
    Assume DeFiLlama reports fee_apr = 5.0%

  incentive_apr:
    DeFiLlama reports incentive_apr = 40.0%

  total_apr = 5.0% + 40.0% = 45.0%

  residual_il_cost = 0.3% (conservative estimate with biweekly rebalancing)

  hedge_funding_cost:
    funding_rate_8h = 0.01% (positive, meaning shorts pay longs)
    funding_rate_annual = 0.0001 * 3 * 365 = 0.1095 = 10.95%
    hedge_ratio = 0.50 (V2 delta)
    hedge_funding_cost = 10.95% * 0.50 = 5.475%

    NOTE: If funding is negative (shorts receive), this becomes income instead of cost!

  rebalance_cost:
    cost_per_rebalance = $75 (gas $5 on Arb + $50 perp trading fee + $20 slippage)
    rebalances_per_year = 24 (biweekly)
    rebalance_cost = ($75 * 24) / $50,000 = $1,800 / $50,000 = 3.6%

  NET YIELD = 5.0% + 40.0% - 0.3% - 5.475% - 3.6% = 35.625% APR

  In USD for 90-day hold:
    income = $50,000 * (5.0% + 40.0%) / 365 * 90 = $5,547.95
    costs = $50,000 * (0.3% + 5.475% + 3.6%) / 365 * 90 = $1,153.42
    net_profit = $5,547.95 - $1,153.42 = +$4,394.53

Caveats:
  - Incentive APR is the dominant factor. Without it, the strategy is NOT profitable.
  - Funding rate can swing wildly. Use 7-day average if available, not just spot rate.
  - Rebalance frequency depends on volatility. High-vol periods need more frequent rebalancing.
  - Gas costs vary by chain. Ethereum mainnet may make this unprofitable for <$200K positions.
```

### 9.4 Funding Rate Cost Formula

```
funding_cost_annual = funding_rate_per_8h * 3 * 365 * short_notional

Where:
  funding_rate_per_8h = current OKX funding rate (e.g. 0.0001 = 0.01%)
  3                   = funding settlements per day (every 8 hours)
  365                 = days per year
  short_notional      = USD value of the perpetual short position

Worked Example:
  funding_rate_per_8h = 0.0001 (0.01%)
  short_notional = $25,000

  funding_cost_annual = 0.0001 * 3 * 365 * $25,000
                      = 0.1095 * $25,000
                      = $2,737.50 per year

  As a percentage of $50,000 LP position:
    $2,737.50 / $50,000 = 5.475%

IMPORTANT: Funding rate sign matters:
  - Positive funding (longs pay shorts): You RECEIVE funding on your short. This is INCOME.
  - Negative funding (shorts pay longs): You PAY funding on your short. This is a COST.
  - Always check the sign. In bull markets, funding is typically positive (good for short hedgers).
  - In bear markets, funding may flip negative (bad for short hedgers -- increases costs).

Funding Rate Scenarios ($25K short, $50K LP):
  +0.03%/8h (bull)  -> +32.85% ann -> -$8,212 income -> +16.42% on LP
  +0.01%/8h (normal) -> +10.95% ann -> -$2,737 income -> +5.48% on LP
  0.00%/8h (flat)    ->  0.00%      -> $0             -> 0.00%
  -0.01%/8h (bear)   -> -10.95% ann -> +$2,737 cost   -> -5.48% on LP
  -0.03%/8h (crash)  -> -32.85% ann -> +$8,212 cost   -> -16.42% on LP
```

### 9.5 Minimum Position Size Formula

```
minimum_position = annual_rebalance_cost / (net_yield_rate - minimum_acceptable_return)

Where:
  annual_rebalance_cost      = cost_per_rebalance * rebalances_per_year
  net_yield_rate             = estimated net yield as a decimal (e.g. 0.35 for 35%)
  minimum_acceptable_return  = 0.01 (1% -- minimum to justify the complexity)

Worked Example:
  cost_per_rebalance = $75
  rebalances_per_year = 24
  annual_rebalance_cost = $75 * 24 = $1,800
  net_yield_rate = 0.35 (35%)

  minimum_position = $1,800 / (0.35 - 0.01)
                   = $1,800 / 0.34
                   = $5,294

  BUT this is the mathematical minimum. In practice, bad months with
  high volatility, negative funding, or incentive reductions can halve yields.

  Practical minimum with 3x safety buffer: $5,294 * 3 = $15,882
  Recommended minimum: $50,000 (accounts for worst-case scenarios
  and ensures rebalance costs remain <5% of total yield)

Position Size Guidelines:
  <$10K: NO (costs >18%), $10-50K: WARN (marginal), $50K-100K: GOOD (recommended min),
  $100K-500K: OPTIMAL, >$500K: EXCELLENT (scale advantage)
```

### 9.6 Rebalance Cost Formula

```
rebalance_cost = lp_adjustment_gas + perp_trading_fee + perp_slippage

Where:
  lp_adjustment_gas = gas cost to adjust LP range or add/remove liquidity
  perp_trading_fee = abs(delta_change) * volatile_value * taker_fee_rate
  perp_slippage = estimated market impact (from orderbook depth)

Per-Chain Gas (LP add/remove): Ethereum $15-$50, Arbitrum $0.30-$1, Base/OP $0.05-$0.20,
  Polygon $0.02-$0.10, BSC $0.10-$0.30, Solana $0.001-$0.01.
  V3 range adjustments cost ~30-50% more than simple add/remove.

Worked Example (Arbitrum, $50K position, price +15%):

  V2 LP: only perp adjustment needed (AMM auto-rebalances)
    gas = $0, trading fee = $0.375, slippage = $0.50 -> total = $0.875

  V3 concentrated: must also re-center LP range
    gas = $1.00, trading fee = $0.75, slippage = $1.00 -> total = $2.75

  Same V3 operation on Ethereum mainnet:
    gas = $40.00, fee = $0.75, slippage = $1.00 -> total = $41.75 (17x more -- L2 is critical)
```

---

## 10. OKX USDT-Margined Swap Fee Schedule (Inlined)

### Perpetual Swap Fees (USDT-Margined)

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.045% |
| VIP2 | >= 10M | 0.010% | 0.040% |
| VIP3 | >= 20M | 0.008% | 0.035% |
| VIP4 | >= 100M | 0.005% | 0.030% |
| VIP5 | >= 200M | 0.002% | 0.025% |

Assume **taker** fees for hedge entry/exit (market orders). Funding fees are separate (Section 9.4).

**Fee formula:** `fee_usd = notional * fee_rate`. Round-trip (VIP0): `$25K * 0.050% * 2 = $25.00 (10 bps)`.

Round-trip taker bps: VIP0=10, VIP1=9, VIP2=8, VIP3=7, VIP4=6, VIP5=5.

---

## 11. Safety Checks

### Pre-Analysis Safety Checklist

Every command runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | -- | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | -- | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 60s stale | > 30s stale | `DATA_STALE` |
| 4 | Pool APY (fees + incentives) | DeFiLlama pool data | < 30% total APY | 30%-35% (thin margin) | `INSUFFICIENT_YIELD` |
| 5 | Pool TVL | DeFiLlama pool data | < $1,000,000 | $1M-$5M (shallow) | `INSUFFICIENT_LIQUIDITY` |
| 6 | Asset has OKX perp | `market_get_instruments(instType: "SWAP")` | No perpetual available | -- | `NO_PERP_AVAILABLE` |
| 7 | Funding rate | `market_get_funding_rate` | -- | > 20% annualized (hedge cost too high) | `FUNDING_TOO_HIGH` |
| 8 | Position size | User input | -- | < $50,000 (rebalance costs may exceed yield) | `POSITION_TOO_SMALL` |
| 9 | GoPlus LP token check | `check_token_security` on LP contract | `isRiskToken === true` or honeypot flags | `is_proxy === "1"` | `SECURITY_BLOCKED` |
| 10 | GoPlus reward token check | `check_token_security` on reward token | Standard BLOCK flags (see below) | Standard WARN flags | `SECURITY_BLOCKED` |
| 11 | Incentive duration | DeFiLlama / protocol info | < 7 days remaining | < 30 days remaining | `INCENTIVE_ENDING` |
| 12 | Net profitability | Full cost calculation | `net_profit <= 0` | `profit_to_cost_ratio < 2` | `NOT_PROFITABLE` |

### GoPlus Security Checks

For every LP opportunity, run `check_token_security` on: (1) LP token contract, (2) reward token contract.

Chain IDs: Ethereum=`"1"`, BSC=`"56"`, Polygon=`"137"`, Arbitrum=`"42161"`, Base=`"8453"`, Avalanche=`"43114"`, Optimism=`"10"`, Solana=`"solana"`.

**BLOCK if any:** `is_honeypot=1`, `buy_tax>5%`, `sell_tax>10%`, `can_take_back_ownership=1`, `owner_change_balance=1`, `is_open_source=0`, `slippage_modifiable=1`, `cannot_sell_all=1`, `top10_holders>80%`.

**WARN if any:** `buy_tax>1%`, `sell_tax>2%`, `is_mintable=1`, `is_proxy=1`, `transfer_pausable=1`, `top10_holders>50%`.

If GoPlus unreachable: `[WARN]` "安全檢查服務暫時無法使用，請謹慎操作。" Never silently skip.

### LP-Hedge-Specific Risk Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| `min_pool_apy_pct` | 30% | 10% (hard floor) | Minimum total APY (fees + incentives) to recommend |
| `min_pool_tvl_usd` | $1,000,000 | $100,000 (hard floor) | Minimum pool TVL |
| `max_position_usd` | $100,000 | $1,000,000 | Maximum recommended position size |
| `max_hedge_leverage` | 1x | 1x | No leverage on the short leg |
| `min_incentive_duration_days` | 30 | 7 | Minimum remaining incentive program duration |
| `max_funding_rate_annual_pct` | 20% | 50% | Maximum acceptable annualized funding rate cost |
| `max_single_pool_allocation_pct` | 25% | 50% | Maximum allocation to any single pool |

### BLOCK Conditions Summary

| Condition | Action | Error Code |
|-----------|--------|------------|
| Pool total APY < 30% | BLOCK. Not profitable after hedge costs. | `INSUFFICIENT_YIELD` |
| Pool TVL < $1M | BLOCK. Insufficient depth for entry/exit. | `INSUFFICIENT_LIQUIDITY` |
| No OKX perp for the volatile asset | BLOCK. Cannot hedge. | `NO_PERP_AVAILABLE` |
| GoPlus flags LP token or reward token | BLOCK. Security risk. | `SECURITY_BLOCKED` |
| Net yield <= 0 after all costs | BLOCK. Not profitable. | `NOT_PROFITABLE` |
| Position size > $1M (hard cap) | BLOCK. Cap at limit. | `POSITION_SIZE_EXCEEDED` |

### WARN Conditions Summary

WARN triggers (flag but allow): position <$50K, funding >20% annual, incentives <30 days remaining, pool APY 30-35% (thin margin), TVL $1-5M, Ethereum mainnet gas, V3 range <±5%. Each WARN must show the specific impact on net yield and be displayed prominently.

---

## 12. Rebalancing Rules (Inlined)

### When to Rebalance the Hedge

The hedge must be adjusted when price moves cause the LP delta to drift from the initial hedge ratio. These rules balance hedge accuracy against rebalance costs.

| Price Change (from entry) | Time Since Last Rebalance | Action |
|--------------------------|--------------------------|--------|
| > 20% | Any | **REBALANCE IMMEDIATELY** -- delta drift is too large |
| > 10% | > 1 hour | **REBALANCE** -- significant drift with sufficient time |
| > 5% | > 6 hours | **REBALANCE** -- moderate drift, avoid over-trading |
| Any | > 24 hours | **REBALANCE** regardless -- keep hedge fresh |
| < 5% | < 6 hours | **WAIT** -- rebalance cost exceeds drift correction value |

### Rebalance Procedure

1. Fetch current price, recalculate delta, compute `new_short = delta * volatile_value / price`
2. Adjustment = `new_short - current_short` (positive = increase short, negative = decrease)
3. For V3: also check if price is near range boundary (may need LP range adjustment)

Worked Example (V2, $10K LP, ETH $3,000->$3,450):
  new_short = 0.50 * $5,000 / $3,450 = 0.725 ETH (was 0.833)
  adjustment = -0.108 ETH, cost = $372.60 * 0.050% + $0.50 slippage = $0.69
  Rule: +15% > 10% AND >1h -> REBALANCE

### V3 Concentrated: Range Exit Scenario

If price exits the V3 range, LP becomes 100% one asset (no fees earned), delta shifts to ~1.0. You are now over-hedged and must immediately reduce the short AND decide whether to re-center the range (extra gas + slippage). This is why V3 concentrated hedging is harder and more expensive.

### Estimated Rebalance Frequency by Volatility Regime

| Volatility (30d annualized) | V2 Rebalances/Month | V3 ±10% Rebalances/Month | V3 ±5% Rebalances/Month |
|-----------------------------|--------------------|--------------------------|-----------------------|
| < 30% (low vol) | 1-2 | 2-3 | 3-5 |
| 30-60% (normal) | 2-3 | 3-5 | 5-8 |
| 60-100% (high vol) | 3-5 | 5-8 | 8-15 |
| > 100% (extreme) | 5-8 | 8-15 | 15-30 |

---

## 13. Output Templates (Inlined)

### Global Header Template

Used at the top of every skill output:

```
══════════════════════════════════════════
  LP Hedge -- Delta-Neutral Yield
  [{MODE}] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

For lp-hedge: Data sources = "OKX REST (perp + funding) + DeFiLlama (pool data) + GoPlus (security)"

### Formatting Rules

- Monetary: `+$1,234.56` or `-$1,234.56` (2 decimals, comma thousands)
- Percentages: `12.5%` (1 decimal), APY/APR: 2 decimal places
- Basis Points: integer only (e.g., `21 bps`)
- Risk Levels: `[SAFE]`, `[WARN]`, `[BLOCK]`
- Risk Gauge: `▓▓▓░░░░░░░ 3/10` (scale 1-10)
- Sparklines: `▁▂▃▄▅▆▇█` for trend visualization
- Timestamps: `YYYY-MM-DD HH:MM UTC`
- Delta: 2 decimal places (e.g., `0.65`)
- Hedge amounts: asset units with 3 decimal places (e.g., `1.083 ETH`)

### Risk Gauge Template

```
── Risk Assessment ─────────────────────────

  Overall Risk:  {RISK_GAUGE}  {RISK_SCORE}/10
                 {RISK_LABEL}

  Breakdown:
  ├─ Smart Contract:   {SC_GAUGE}    {SC_SCORE}/10
  │                    {SC_NOTE}
  ├─ Funding Rate:     {FR_GAUGE}    {FR_SCORE}/10
  │                    {FR_NOTE}
  ├─ IL Residual:      {IL_GAUGE}    {IL_SCORE}/10
  │                    {IL_NOTE}
  ├─ Incentive Risk:   {IR_GAUGE}    {IR_SCORE}/10
  │                    {IR_NOTE}
  └─ Liquidity Risk:   {LIQ_GAUGE}   {LIQ_SCORE}/10
                       {LIQ_NOTE}
```

Gauge format: `1/10: ▓░░░░░░░░░`, `5/10: ▓▓▓▓▓░░░░░`, `10/10: ▓▓▓▓▓▓▓▓▓▓`
Risk labels: 1-2 LOW RISK, 3-4 MODERATE-LOW, 5-6 MODERATE, 7-8 ELEVATED, 9-10 HIGH RISK

### Risk Score Components (LP-Hedge Specific)

| Component | Weight | Scoring |
|-----------|--------|---------|
| Smart contract risk | 0-2 | 0 = top protocol + audited, 1 = audited, 2 = unaudited/new |
| Funding rate volatility | 0-2 | 0 = consistently positive, 1 = variable, 2 = frequently negative |
| IL residual risk | 0-2 | 0 = stable pair, 1 = V2 volatile pair, 2 = V3 concentrated volatile |
| Incentive sustainability | 0-2 | 0 = >90 days + established, 1 = 30-90 days, 2 = <30 days or new |
| Pool liquidity depth | 0-2 | 0 = TVL >$50M, 1 = $5M-$50M, 2 = <$5M |

### 13.1 Scan Output

```
══════════════════════════════════════════
  LP Hedge -- SCAN
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:00 UTC
  Data sources: OKX REST + DeFiLlama + GoPlus
══════════════════════════════════════════

── 可對沖 LP 池掃描結果 ─────────────────────

  #1  ETH-USDC 0.3% (Uniswap V3 / Arbitrum)
  ├─ TVL:              $45,200,000
  ├─ 費用 APR:         5.20%
  ├─ 激勵 APR:         42.00% (ARB rewards, 65 天剩餘)
  ├─ 總 APR:           47.20%
  ├─ OKX Perp:         ETH-USDT-SWAP [SAFE]
  ├─ 資金費率 (年化):   +8.50% (做空收取)
  ├─ 預估淨收益:        38.30%
  ├─ 安全檢查:          [SAFE] GoPlus verified
  └─ 風險:             ▓▓▓░░░░░░░ 3/10

  #2  AVAX-USDC 0.3% (Trader Joe / Avalanche)
  ├─ TVL:              $3,200,000
  ├─ 費用 APR:         3.50%
  ├─ 激勵 APR:         38.00% (JOE rewards, 20 天剩餘)
  ├─ 總 APR:           41.50%
  ├─ OKX Perp:         AVAX-USDT-SWAP [SAFE]
  ├─ 資金費率 (年化):   +6.20% (做空收取)
  ├─ 預估淨收益:        31.20%
  ├─ 安全檢查:          [WARN] 激勵計劃剩餘不足 30 天
  └─ 風險:             ▓▓▓▓▓▓░░░░ 6/10

──────────────────────────────────────────
  已掃描 142 個池 | 符合條件: 2 個
  篩選: APY >= 30%, TVL >= $1M, OKX 永續合約可用
```

### 13.2 Evaluate Output

```
══════════════════════════════════════════
  LP Hedge -- EVALUATE
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX REST + DeFiLlama + GoPlus
══════════════════════════════════════════

── ETH-USDC 0.3% on Uniswap V3 (Arbitrum) ──

  LP 類型:            V3 Concentrated (±10%)
  倉位規模:           $50,000.00
  持有期:             90 天

── 市場數據 ──────────────────────────────

  ETH 現價:           $3,000.00
  ETH-USDT-SWAP:      資金費率 +0.01% / 8h (年化 +10.95%)
  池 TVL:             $45,200,000
  池 24h 交易量:       $8,500,000

── 對沖參數 ──────────────────────────────

  LP Delta:           0.65 (V3 ±10% concentrated)
  ETH in LP:          $25,000.00 (8.333 ETH)
  Short ETH:          5.417 ETH ($16,250.00)
  保證金需求:          $16,250.00 (1x, 全額抵押)
  總資金需求:          $50,000 (LP) + $16,250 (margin) = $66,250

── 收益分析 (90 天) ─────────────────────

  收入:
  ├─ LP 費用收入:     +$616.44   (5.0% APR * $50K * 90/365)
  ├─ 激勵獎勵:        +$4,931.51  (40.0% APR * $50K * 90/365)
  ├─ 資金費率收入:     +$438.70   (10.95% * $16,250 * 90/365, 正 funding = 做空收取)
  └─ 小計:            +$5,986.65

  成本:
  ├─ 殘餘 IL:          -$36.99   (0.3% * $50K * 90/365)
  ├─ 再平衡成本:       -$150.00  (~$25/次 * 6次, Arbitrum L2)
  ├─ LP 進場 Gas:      -$1.00    (Arbitrum)
  ├─ 對沖進場手續費:   -$8.13    ($16,250 * 0.050%)
  ├─ LP 退場成本:      -$2.50    (gas + slippage)
  ├─ 對沖平倉手續費:   -$8.13    ($16,250 * 0.050%)
  └─ 小計:            -$206.74

  ───────────────────────────────────
  淨利潤:             +$5,779.91
  年化淨收益:          46.87% APR
  損益平衡:            3.2 天
  利潤/成本比:         27.95x

── 情境分析 ──────────────────────────────

  情境               | ETH 價格   | 淨 PnL    | 年化淨收益
  ───────────────────┼───────────┼──────────┼──────────
  價格不變 (±0%)     | $3,000    | +$5,779  | 46.87%
  價格 +20%          | $3,600    | +$5,350  | 43.39%
  價格 -20%          | $2,400    | +$5,280  | 42.82%
  價格 +50%          | $4,500    | +$4,150  | 33.66%
  價格 -50%          | $1,500    | +$3,890  | 31.54%

  NOTE: 價格大幅波動時需更頻繁 rebalance，成本上升，收益略降。
  上述已計入殘餘 IL 和額外 rebalance 成本估算。

── 安全檢查 & 風險 ──────────────────────

  [SAFE] OKX MCP, LP Token, ARB Token, ETH-USDT-SWAP -- all verified
  [WARN] 資金費率 +10.95% 年化 -- 目前做空有利，但可能轉負
  Overall: [SAFE]  ▓▓▓░░░░░░░ 3/10 MODERATE-LOW

── Recommendation ────────────────────────

  建議執行：年化 46.87%，3.2 天損益平衡，Arbitrum gas 極低
  注意：激勵結束後降至 ~5%，需監控資金費率，每 1-2 週 rebalance

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════
  1. lp-hedge hedge-calc --asset ETH --size-usd 50000 --lp-type v3_concentrated --range-pct 10
  2. 手動: LP $50K (Arb Uni V3 ±10%) + OKX short 5.417 ETH ($16,250 margin)
  3. lp-hedge rebalance-check --asset ETH --entry-price 3000 --short-amount 5.417 ...

  ── Disclaimer ────────────────────────────
  以上僅為分析建議，不會自動執行任何交易。所有建議均需人工審核後手動操作。
  過去收益及資金費率不代表未來表現。激勵計劃可能提前結束或無預警變更。
══════════════════════════════════════════
```

### 13.3 Hedge-Calc Output

Standard header + hedge parameters (delta, short amount, margin), rebalance trigger table (±20%/±10%/±5%/24h rules), cost per rebalance breakdown, annual rebalance cost estimate. See Section 12 for trigger rules and Section 9.6 for cost formulas.

### 13.4 Rebalance-Check Output

Standard header + current vs required hedge comparison, price change %, adjustment direction and amount, decision (`REBALANCE_NOW`/`WAIT`/`NO_CHANGE`), triggered rule, adjustment cost, and step-by-step instructions.

Every output ends with: Next Steps (1-3 actionable commands), Related Commands, and the standard Disclaimer (analysis only, no auto-execution, bilingual ZH/EN).

---

## 14. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify config, restart server. |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗，請檢查 OKX API 金鑰設定 | API authentication failed. Check OKX API key configuration. | Update `~/.okx/config.toml` |
| `DEFI_DATA_UNAVAILABLE` | DeFiLlama MCP unreachable | DeFi 池數據服務無法連線。LP 掃描功能不可用。 | DeFi pool data service unreachable. LP scan unavailable. | Check DeFiLlama MCP status. Cannot degrade gracefully -- DeFi data is essential. |
| `SECURITY_UNAVAILABLE` | GoPlus MCP unreachable | 安全檢查服務暫時無法使用，請謹慎操作 | Security check service unavailable. Proceed with extreme caution. | Retry once, then WARN and continue with disclaimer. |
| `NO_PERP_AVAILABLE` | No OKX perpetual swap for the volatile asset | {asset} 在 OKX 沒有永續合約，無法對沖 | No perpetual swap available for {asset} on OKX. Cannot hedge. | Suggest alternative assets with perps. |
| `INSUFFICIENT_YIELD` | Pool total APY < 30% | 池 APY ({apy}%) 低於 30% 門檻，扣除對沖成本後不具盈利性 | Pool APY ({apy}%) below 30% threshold. Not profitable after hedge costs. | Show full cost breakdown. Suggest yield-optimizer for non-hedged options. |
| `INSUFFICIENT_LIQUIDITY` | Pool TVL < $1M | 池 TVL (${tvl}) 不足 $1M，深度不足以安全進出場 | Pool TVL (${tvl}) below $1M minimum. Insufficient depth for safe entry/exit. | Suggest larger pools or different chains. |
| `INCENTIVE_ENDING` | Incentive program ending soon | 激勵計劃剩餘 {days} 天。結束後淨收益可能為負。 | Incentive program ends in {days} days. Net yield may turn negative after. | Calculate post-incentive net yield. If negative, BLOCK. |
| `FUNDING_TOO_HIGH` | Funding rate > 20% annualized as a cost | 資金費率年化 {rate}% 偏高，對沖成本過大 | Funding rate annualized {rate}% is high. Hedge cost excessive. | Show net yield with and without hedge. Consider partial hedge. |
| `POSITION_TOO_SMALL` | Position < $50K | 倉位 ${size} 低於建議最低 $50K，再平衡成本佔比偏高 | Position ${size} below recommended $50K minimum. Rebalance costs disproportionate. | Show rebalance cost as % of position. Allow if user insists. |
| `POSITION_SIZE_EXCEEDED` | Position > $1M hard cap | 倉位金額 ${amount} 超過上限 $1,000,000 | Position size ${amount} exceeds $1,000,000 limit. | Cap at limit. |
| `INSTRUMENT_NOT_FOUND` | OKX instrument query returns empty | 找不到指定的交易工具 | Specified instrument not found on OKX. | Check instrument ID format. |
| `DATA_STALE` | Market data > 60s old | 市場數據已過期，正在重新獲取... | Market data stale. Refetching... | Auto-retry once, then error. |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. | Wait 1s, retry up to 3x. |
| `NOT_PROFITABLE` | Net P&L is zero or negative | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行 | Net profit is negative after all costs ({net_pnl}). Not recommended. | Show full cost breakdown. |
| `SECURITY_BLOCKED` | GoPlus flags LP token or reward token | 安全檢查未通過：{reason}。已從推薦中移除。 | Security check failed: {reason}. Removed from recommendations. | Show specific GoPlus findings. Do not proceed. |

---

## Conversation Examples

### Example 1: "有冇得做 LP 挖礦順便對沖嘅？"
**Command:** `scan` -> DeFiLlama pools (>30% APY, >$1M TVL) x OKX perps -> GoPlus check -> rank by net yield -> output (13.1)

### Example 2: "ETH-USDC LP on Uniswap V3 Arbitrum 值唔值得做 delta neutral？"
**Command:** `evaluate --pool "ETH-USDC" --protocol "Uniswap V3" --chain arbitrum` -> pool data + spot + funding + orderbook + GoPlus -> full projection + scenarios -> output (13.2)

### Example 3: "我想知要 short 幾多 ETH 對沖 $10,000 V2 LP"
**Command:** `hedge-calc --asset ETH --size-usd 10000 --lp-type v2` -> price -> delta=0.50 -> short=0.833 ETH -> rebalance triggers -> output (13.3)

### Example 4: "ETH 升咗 15%，我嘅 hedge 要唔要調？"
**Command:** `rebalance-check --asset ETH --entry-price 3000 --short-amount 0.833 --size-usd 10000 --lp-type v2 --hours-since-last 8` -> +15% > 10% AND 8h > 1h -> REBALANCE -> output (13.4)

---

## Cross-Reference

| Reference | File | Sections Used |
|-----------|------|---------------|
| Formulas | `references/formulas.md` | IL formula, funding rate annualization |
| Fee Schedule | `references/fee-schedule.md` | OKX swap fees, gas benchmarks |
| Safety Checks | `references/safety-checks.md` | Full checklist, GoPlus decision matrix |
| Output Templates | `references/output-templates.md` | Header, risk gauge, next steps |
| Risk Limits | `config/risk-limits.example.yaml` | `lp_hedge` section |
| GoPlus Tools | `references/goplus-tools.md` | `check_token_security` for LP and reward tokens |
| Agent Orchestration | `AGENTS.md` | LP hedging chain |

> **Note:** All content from the above reference files has been inlined in this document for OpenClaw/Lark compatibility. The LLM does not need to read any external files.
