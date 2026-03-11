---
name: profitability-calculator
description: >
  Calculates net profitability of proposed trades after ALL cost layers: trading fees,
  gas costs, slippage, withdrawal fees, bridge fees, funding payments, and borrow costs.
  Single source of truth for "is this trade profitable?"
  Activates when user asks: "is this profitable", "賺唔賺", "cost analysis", "成本分析",
  "how much would I make", "能賺多少", "fee breakdown", "費用明細", "minimum spread",
  "最小價差", "break-even", "盈虧平衡".
  Do NOT use for: price discovery (use price-feed-aggregator), trade execution,
  token security (use GoPlus), or strategy-specific analysis (use strategy skills).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:system_get_capabilities
---

# profitability-calculator

## 1. Role

Net P&L engine for the entire Onchain x CEX Strats skill suite. **No strategy skill should
recommend a trade without passing through this calculator first.** This is the single gate
that determines whether an opportunity is worth executing.

### What This Skill Computes

- **Gross spread / profit** between trade legs
- **All cost layers** itemized individually
- **Net profit** after subtracting every cost
- **Profit-to-cost ratio** as a quality metric
- **Minimum spread for breakeven** given the cost structure
- **Sensitivity analysis** across variable ranges (size, slippage, gas)

### What This Skill Does NOT Do

- **Execute trades** -- analysis only, never sends orders
- **Fetch prices** -- expects prices as input from `price-feed-aggregator` or caller
- **Check token security** -- delegate to GoPlus MCP
- **Discover opportunities** -- delegate to strategy skills (`cex-dex-arbitrage`, etc.)
- **Manage positions** -- no state is persisted between calls

---

## 2. Language

Match the user's language. Default: **Traditional Chinese (繁體中文)**.

Technical labels may remain in English regardless of language:
- PnL, APY, bps, USD, ETH, gas, gwei, slippage, taker, maker

Examples:
- User writes English --> respond in English
- User writes "賺唔賺" --> respond in Traditional Chinese
- User writes "能赚多少" --> respond in Simplified Chinese

---

## 3. Account Safety

### Demo Default

- Always use `okx-DEMO-simulated-trading` unless the user explicitly requests live mode.
- Every output header includes `[DEMO]` or `[LIVE]`.
- Every output header includes `[RECOMMENDATION ONLY -- 不會自動執行]`.

### Switching to Live

1. User explicitly says "live", "真實帳戶", "real account", or equivalent.
2. Display confirmation warning (bilingual).
3. Wait for user to reply "確認" or "confirm".
4. Call `system_get_capabilities` to verify `authenticated: true`.
5. If authenticated: switch. If not: show `AUTH_FAILED`, remain in demo.

### Switching to Demo

Immediate, no confirmation needed. User says "demo" or "模擬".

---

## 4. Pre-flight

This skill has minimal dependencies. It needs:

| Dependency | Purpose | Required? |
|-----------|---------|-----------|
| `okx-trade-mcp` | Fee schedule lookup, orderbook depth for slippage estimation | Yes |
| `OnchainOS CLI` | Gas price fetch (`onchain-gateway gas`), gas limit estimation | Only for onchain legs |
| Fee tables (inlined below) | Static fee tables, withdrawal fee lookup | Yes (built-in) |
| Formulas (inlined below) | Calculation formulas | Yes (built-in) |

### Pre-flight Checks

```
1. Call system_get_capabilities
   - Verify authenticated: true
   - Note mode: "demo" or "live"

2. If trade has onchain legs:
   - Verify OnchainOS CLI is available: `which onchainos`
   - If unavailable: use benchmark gas values from fee tables below
     and flag confidence as "medium"
```

---

## 5. Skill Routing Matrix

| User Need | THIS Skill Handles | Delegate To |
|-----------|-------------------|-------------|
| "Is this arb profitable?" | Estimate with all legs and costs | -- |
| "What are the fees on OKX?" | Full fee breakdown per leg | -- |
| "Minimum spread needed?" | Compute min-spread for breakeven | -- |
| "How does profit change with size?" | Sensitivity analysis | -- |
| "What's the current price?" | -- | `price-feed-aggregator` |
| "Find arb opportunities" | -- | `cex-dex-arbitrage` |
| "Is this token safe?" | -- | GoPlus MCP directly |
| "What's the funding rate?" | -- | `funding-rate-arbitrage` |
| "Execute this trade" | -- | Not supported (analysis only) |

---

## 6. Command Index

| Command | Purpose | Typical Caller |
|---------|---------|---------------|
| `estimate` | Compute net P&L for a set of trade legs | Strategy skills, user |
| `breakdown` | Detailed per-layer cost table | User (interactive) |
| `min-spread` | Minimum spread (bps) for breakeven between two venues | Strategy skills |
| `sensitivity` | How net profit varies across a parameter range | User (analysis) |

---

## 7. Parameter Reference

### 7.1 Command: `estimate`

Compute net profit/loss for a proposed multi-leg trade.

#### Usage

```bash
profitability-calculator estimate --legs '[
  {"venue":"okx-cex","asset":"ETH","side":"buy","size_usd":1000,"order_type":"market"},
  {"venue":"okx-dex","chain":"ethereum","asset":"ETH","side":"sell","size_usd":1000,"order_type":"market"}
]'
```

#### Input Schema: TradeLeg

```yaml
TradeLeg:
  venue: string           # REQUIRED. "okx-cex" | "okx-dex" | "uniswap" | "jupiter"
  chain: string           # REQUIRED for DEX legs. "ethereum" | "solana" | "base" | "arbitrum" etc.
  asset: string           # REQUIRED. Symbol (e.g. "ETH") or contract address
  side: string            # REQUIRED. "buy" | "sell"
  size_usd: number        # REQUIRED. Trade size in USD
  order_type: string      # Optional. "market" (default) | "limit"
  estimated_slippage_bps: number  # Optional. Default: auto-calculated from orderbook
  gas_gwei: number        # Optional. For onchain legs. Default: fetched from onchain-gateway
  withdrawal_required: boolean    # Optional. Default: false
  deposit_required: boolean       # Optional. Default: false
  bridge_required: boolean        # Optional. Default: false
  bridge_chain: string           # Required if bridge_required=true. Target chain.
  vip_tier: string               # Optional. "VIP0" (default) | "VIP1" ... "VIP5"
  instrument_type: string        # Optional. "spot" (default) | "swap" | "futures"
  funding_intervals: number      # Optional. For perp legs. Number of 8h funding intervals held.
  funding_rate: number           # Optional. Current funding rate per 8h (e.g. 0.00015)
  borrow_rate_annual: number     # Optional. Annualized borrow rate for margin/short legs.
  hold_days: number              # Optional. Expected holding period in days. Default: 0 (instant).
```

#### Parameter Validation

| Field | Type | Validation | On Failure |
|-------|------|-----------|------------|
| `venue` | string | Must be one of: `okx-cex`, `okx-dex`, `uniswap`, `jupiter`, `curve`, `pancakeswap` | `INVALID_LEG` |
| `chain` | string | Required when venue is not `okx-cex`. Must match Supported Chains Table below. | `MISSING_FIELD` |
| `asset` | string | Non-empty. If contract address, must be lowercase for EVM chains. | `MISSING_FIELD` |
| `side` | string | Must be `buy` or `sell` | `INVALID_LEG` |
| `size_usd` | number | Must be > 0 | `INVALID_LEG` |
| `order_type` | string | Must be `market` or `limit`. Default: `market` | `INVALID_LEG` |
| `estimated_slippage_bps` | number | Must be >= 0. If omitted, auto-calculated. | -- |
| `gas_gwei` | number | Must be >= 0. If omitted, fetched live or from benchmark. | -- |
| `bridge_chain` | string | Required if `bridge_required=true` | `MISSING_FIELD` |
| `vip_tier` | string | Must be `VIP0`..`VIP5`. Default: `VIP0` | `INVALID_LEG` |
| `instrument_type` | string | Must be `spot`, `swap`, or `futures`. Default: `spot` | `INVALID_LEG` |
| `funding_intervals` | number | Must be >= 0. Only applies when instrument_type=`swap` | -- |
| `funding_rate` | number | If omitted and funding_intervals > 0, fetched live via `market_get_funding_rate` | `FEE_LOOKUP_FAILED` |
| `borrow_rate_annual` | number | Must be >= 0. Only applies to margin/short legs. | -- |
| `hold_days` | number | Must be >= 0. Default: 0 (instant execution). | -- |

#### Return Schema: ProfitabilityResult

```yaml
ProfitabilityResult:
  net_profit_usd: number         # Positive = profitable, negative = loss
  gross_spread_usd: number       # Before any costs
  total_costs_usd: number        # Sum of all cost layers
  cost_breakdown:
    cex_trading_fee: number      # size_usd * fee_rate (per CEX leg)
    dex_trading_fee: number      # From swap quote or estimated from protocol fee table
    gas_cost: number             # For onchain legs: gas_gwei * gas_limit * native_price / 1e9
    slippage_cost: number        # Estimated from orderbook depth or user-provided bps
    withdrawal_fee: number       # Fixed per-asset from OKX schedule
    deposit_fee: number          # Usually 0 (user pays network gas only)
    bridge_fee: number           # If cross-chain transfer required
    funding_payment: number      # For perp legs held over funding intervals
    borrow_cost: number          # For margin/short legs: size * borrow_rate * hold_days / 365
  profit_to_cost_ratio: number   # net_profit / total_costs (undefined if total_costs = 0)
  is_profitable: boolean         # true if net_profit_usd > 0
  min_spread_for_breakeven_bps: number  # Minimum spread needed to cover all costs
  confidence: string             # "high" | "medium" | "low"
  confidence_factors:
    - factor: string             # Description of what affects confidence
      impact: string             # "reduces_confidence" | "increases_confidence"
  warnings: string[]             # Array of warning messages
```

#### Confidence Levels

| Level | Criteria |
|-------|---------|
| `high` | All cost data fetched live (gas, orderbook, fees). Slippage auto-calculated. |
| `medium` | Some data from benchmarks (e.g. gas from fee tables, not live). |
| `low` | User-provided estimates for multiple fields. No live data verification. |

---

### 7.2 Command: `breakdown`

Same input as `estimate`, but outputs a detailed per-layer table with additional context
per cost item (formula used, data source, alternative options).

#### Usage

```bash
profitability-calculator breakdown --legs '[...]'
```

#### Additional Output Fields (beyond ProfitabilityResult)

Each cost line includes:

```yaml
cost_detail:
  label: string          # Human-readable label (e.g. "CEX Taker Fee (VIP0)")
  amount_usd: number     # Cost in USD
  amount_bps: number     # Cost in basis points relative to trade size
  pct_of_gross: number   # This cost as % of gross spread
  formula: string        # The formula used (e.g. "1000 * 0.0008 = $0.80")
  data_source: string    # Where the input came from ("live orderbook", "fee tables", "user input")
  optimization_hint: string  # Optional suggestion (e.g. "Use limit order for maker fee: 0.06%")
```

---

### 7.3 Command: `min-spread`

Calculate the minimum spread (in bps) required between two venues for a trade to be
profitable after all costs.

#### Usage

```bash
profitability-calculator min-spread \
  --venue-a okx-cex \
  --venue-b okx-dex \
  --chain ethereum \
  --asset ETH \
  --size-usd 10000 \
  --vip-tier VIP0 \
  --order-type market
```

#### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| `--venue-a` | Yes | -- | string | First venue (buy side) |
| `--venue-b` | Yes | -- | string | Second venue (sell side) |
| `--chain` | Conditional | -- | string | Required if either venue is DEX |
| `--asset` | Yes | -- | string | Asset symbol or contract address |
| `--size-usd` | Yes | -- | number | Trade size in USD |
| `--vip-tier` | No | `VIP0` | string | OKX VIP tier |
| `--order-type` | No | `market` | string | `market` or `limit` |
| `--include-withdrawal` | No | `true` | boolean | Include withdrawal fee in cost |
| `--include-bridge` | No | `false` | boolean | Include bridge fee in cost |
| `--bridge-chain` | Conditional | -- | string | Required if include-bridge=true |

#### Return Schema

```yaml
MinSpreadResult:
  min_spread_bps: number         # Minimum spread needed (breakeven point)
  cost_components_bps:
    cex_fee_bps: number
    dex_gas_bps: number
    dex_fee_bps: number
    slippage_bps: number
    withdrawal_bps: number
    bridge_bps: number
  total_cost_bps: number
  interpretation: string         # e.g. "Need at least 18 bps spread to break even on $10,000"
```

---

### 7.4 Command: `sensitivity`

Show how net profit changes across a range of values for a given variable.

#### Usage

```bash
profitability-calculator sensitivity \
  --legs '[...]' \
  --variable size_usd \
  --range '[1000, 50000]' \
  --steps 10
```

#### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| `--legs` | Yes | -- | TradeLeg[] | Base trade configuration |
| `--variable` | Yes | -- | string | Variable to sweep: `size_usd`, `slippage`, `gas_gwei`, `vip_tier`, `spread_bps` |
| `--range` | Yes | -- | [min, max] | Range of values to test |
| `--steps` | No | `10` | number | Number of data points in the range |

#### Return Schema

```yaml
SensitivityResult:
  variable: string
  data_points:
    - value: number              # Variable value at this point
      net_profit_usd: number
      total_costs_usd: number
      profit_to_cost_ratio: number
      is_profitable: boolean
  breakeven_value: number        # Value of variable where net_profit = 0
  optimal_value: number          # Value with highest profit_to_cost_ratio
  summary: string                # Human-readable interpretation
```

---

## 8. Operation Flow

### Step 1: Parse and Validate

```
INPUT: TradeLeg[]
  |
  ├─ Validate all REQUIRED fields present
  ├─ Validate field values against allowed enums
  ├─ Validate cross-field dependencies:
  │   - DEX venue requires `chain`
  │   - bridge_required=true requires `bridge_chain`
  │   - funding_intervals > 0 requires instrument_type="swap"
  |
  ├─ On validation failure → return INVALID_LEG or MISSING_FIELD
  └─ On success → proceed to Step 2
```

### Step 2: Fetch Live Cost Data

For each leg, gather the cost inputs:

```
FOR EACH leg IN TradeLeg[]:

  IF leg.venue == "okx-cex":
    ├─ Fee rate: lookup from OKX Fee Tables below using vip_tier + instrument_type
    │   - Spot VIP0 taker: 0.080%, maker: 0.060%
    │   - Swap VIP0 taker: 0.050%, maker: 0.020%
    │
    ├─ Slippage: if not provided, fetch orderbook:
    │   market_get_orderbook(instId={ASSET}-USDT, sz="400")
    │   Walk bids/asks to compute volume-weighted avg price at size_usd
    │   price_impact_bps = (vwap - best_price) / best_price * 10000
    │
    └─ Withdrawal fee: if withdrawal_required=true
        Lookup from OKX Withdrawal Fee Tables below → fixed amount per asset+network

  IF leg.venue is DEX (okx-dex, uniswap, jupiter, etc.):
    ├─ Gas price: if gas_gwei not provided:
    │   onchainos onchain-gateway gas --chain {chain}
    │   On failure → use benchmark from Gas Benchmarks table below, set confidence="medium"
    │
    ├─ Gas limit: use benchmarks from Gas Benchmarks table below:
    │   - Ethereum: 150,000-300,000
    │   - Arbitrum: 1,000,000-2,000,000
    │   - Solana: N/A (use lamport-based calculation)
    │   Or fetch via onchainos onchain-gateway gas-limit if calldata available
    │
    ├─ DEX protocol fee: lookup from DEX Protocol Fee Tables below:
    │   - Uniswap standard: 30 bps
    │   - Jupiter: 0 bps (aggregator layer)
    │   - Note: DEX quotes from aggregators already include protocol fees
    │     so set dex_trading_fee = 0 when using aggregator quotes
    │
    └─ Slippage: if not provided:
        Use price impact from dex-swap quote if available
        Otherwise estimate from benchmarks

  IF leg has funding_intervals > 0:
    ├─ If funding_rate not provided:
    │   market_get_funding_rate(instId={ASSET}-USDT-SWAP)
    │   Use fundingRate field
    │
    └─ funding_payment = size_usd * funding_rate * funding_intervals
        (positive if receiving, negative if paying)

  IF leg has borrow_rate_annual > 0 AND hold_days > 0:
    └─ borrow_cost = size_usd * borrow_rate_annual * hold_days / 365
```

### Step 3: Compute Each Cost Layer and Net Profit

```
# Formulas (all inlined below in Section 9)

cex_trading_fee = size_usd * fee_rate
  - fee_rate: 0.0008 (taker VIP0 spot) or 0.0006 (maker VIP0 spot)
  - For swaps: 0.0005 (taker VIP0) or 0.0002 (maker VIP0)

gas_cost = gas_gwei * gas_limit * native_token_price_usd / 1e9
  - Example: 30 gwei * 150,000 * $3,450 / 1e9 = $15.53
  - Solana: (base_fee_lamports + priority_fee_lamports) * sol_price / 1e9

slippage_cost = size_usd * estimated_slippage_bps / 10000
  - Auto-calc: walk orderbook to find VWAP at size_usd
  - price_impact_bps = (vwap - best_price) / best_price * 10000

withdrawal_fee = lookup from OKX Withdrawal Fee Tables below
  - ETH (ERC-20): 0.00035 ETH
  - ETH (Arbitrum): 0.0001 ETH
  - USDT (TRC-20): 1.0 USDT
  - USDT (Arbitrum): 0.1 USDT

bridge_fee = estimated from Bridge Fee Tables below
  - Across Protocol: 0.04-0.12% of amount
  - OKX Bridge: 0-0.1%

funding_payment = size_usd * funding_rate * funding_intervals
  - Positive funding_rate + short position = receive payment
  - Positive funding_rate + long position = pay funding

borrow_cost = size_usd * borrow_rate_annual * hold_days / 365

# Aggregation
total_costs = SUM(cex_trading_fee, dex_trading_fee, gas_cost,
                  slippage_cost, withdrawal_fee, deposit_fee,
                  bridge_fee, funding_payment, borrow_cost)

gross_spread = (sell_price - buy_price) / buy_price * size_usd
  - Or: provided by caller as price differential

net_profit = gross_spread - total_costs
profit_to_cost_ratio = net_profit / total_costs
min_spread_for_breakeven_bps = total_costs / size_usd * 10000
```

### Step 4: Format Output and Flag Results

```
IF net_profit <= 0:
  → Label: [NOT PROFITABLE]
  → Show full cost breakdown so user understands why
  → Suggest: reduce size, switch chain (lower gas), use limit orders, wait for wider spread

IF net_profit > 0 AND profit_to_cost_ratio < 2.0:
  → Label: [MARGINAL]
  → Warning: "利潤率偏低，執行風險大 / Thin margin, high execution risk"

IF net_profit > 0 AND profit_to_cost_ratio >= 2.0:
  → Label: [PROFITABLE]

ALWAYS:
  → Show cost breakdown table
  → Show confidence level and factors
  → Show next steps / related commands
  → Include disclaimer (bilingual)
```

---

## 9. Cost Computation Formulas

### CEX-DEX Spread (basis points)

```
spread_bps = abs(cex_price - dex_price) / min(cex_price, dex_price) * 10000
```

| Variable | Definition |
|----------|-----------|
| `cex_price` | Mid-price on CEX orderbook (OKX), or best bid/ask depending on direction |
| `dex_price` | Quote price returned by DEX aggregator for the given size |
| `spread_bps` | Unsigned spread in basis points (1 bp = 0.01%) |

**Worked Example:**

```
cex_price = 3,412.50  (ETH-USDT mid on OKX)
dex_price = 3,419.80  (Jupiter quote, selling ETH for USDT on Solana)

spread_bps = abs(3412.50 - 3419.80) / min(3412.50, 3419.80) * 10000
           = 7.30 / 3412.50 * 10000
           = 21.4 bps
```

**Caveats:**
- Always use the same quote direction (both buy or both sell) for a fair comparison.
- DEX price already includes DEX protocol fee and price impact for the quoted size.
- CEX price should use bid for sell, ask for buy -- not mid -- when evaluating executable spread.

### Effective Price After Slippage

```
effective_price = quoted_price * (1 + slippage_bps / 10000)   # for buys
effective_price = quoted_price * (1 - slippage_bps / 10000)   # for sells
```

### Price Impact Estimation from Orderbook Depth

```
price_impact_bps = (execution_avg_price - best_price) / best_price * 10000
```

Where `execution_avg_price` is the volume-weighted average price across consumed levels:

```
execution_avg_price = sum(level_price_i * level_qty_i) / sum(level_qty_i)
    for levels consumed until order_size is filled
```

**Worked Example (Selling 10 ETH):**

```
Orderbook bids:
  Level 1: 3,412.50 x 3 ETH
  Level 2: 3,412.00 x 4 ETH
  Level 3: 3,411.20 x 5 ETH

execution_avg_price = (3412.50*3 + 3412.00*4 + 3411.20*3) / (3+4+3)
                    = (10237.50 + 13648.00 + 10233.60) / 10
                    = 34119.10 / 10
                    = 3,411.91

price_impact_bps = (3412.50 - 3411.91) / 3412.50 * 10000
                 = 0.59 / 3412.50 * 10000
                 = 1.7 bps
```

### CEX Trading Fee

```
cex_fee = size_usd * fee_rate
```

| Variable | Definition |
|----------|-----------|
| `size_usd` | Notional trade size in USD |
| `fee_rate` | Maker or taker rate per VIP tier (see fee tables below) |

**Worked Example (VIP1 Taker, $50,000 trade):**

```
cex_fee = 50000 * 0.0007
        = $35.00
```

### DEX Gas Cost (EVM)

```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price_usd / 1e9
```

| Variable | Definition |
|----------|-----------|
| `gas_price_gwei` | Current gas price in gwei (for EVM chains) |
| `gas_limit` | Gas units required for the transaction |
| `native_token_price_usd` | Current USD price of the chain's native token (ETH, BNB, MATIC, etc.) |

**Worked Example (Uniswap swap on Ethereum):**

```
gas_price_gwei = 30
gas_limit = 150,000
native_token_price_usd = 3,400

gas_cost_usd = 30 * 150000 * 3400 / 1e9
             = 15,300,000,000 / 1,000,000,000
             = $15.30
```

### DEX Gas Cost (Solana)

Solana uses lamports instead of gwei:

```
gas_cost_usd = (base_fee_lamports + priority_fee_lamports) * sol_price_usd / 1e9
```

**Worked Example:**

```
tx_fee_lamports = 5,000  (base fee)
priority_fee_lamports = 50,000  (typical priority fee)
sol_price_usd = 145

gas_cost_usd = (5000 + 50000) * 145 / 1e9
             = 7,975,000 / 1,000,000,000
             = $0.008
```

**Gas Caveats:**
- Gas prices are highly volatile, especially on Ethereum mainnet.
- Complex swaps (multi-hop, multi-pool) consume more gas.
- L2s (Arbitrum, Base) also pay an L1 data posting fee that fluctuates.
- Always fetch real-time gas estimates; benchmarks are rough guides only.

### Slippage Cost

```
slippage_cost_usd = size_usd * estimated_slippage_bps / 10000
```

**Worked Example:**

```
size_usd = 50,000
estimated_slippage_bps = 8

slippage_cost_usd = 50000 * 8 / 10000
                  = $40.00
```

**Auto-calculation from orderbook:**

```
1. Fetch orderbook: market_get_orderbook(instId, sz="400")
2. Walk levels until cumulative size >= order_size
3. vwap = sum(price_i * qty_i) / sum(qty_i)
4. price_impact_bps = abs(vwap - best_price) / best_price * 10000
5. slippage_cost = size_usd * price_impact_bps / 10000
```

### Total Cost Aggregation

```
total_cost = cex_fee + dex_gas_cost + dex_protocol_fee + slippage_cost + bridge_fee + withdrawal_fee
```

For a round-trip trade:

```
total_cost_roundtrip = (cex_fee_entry + cex_fee_exit)
                     + (dex_gas_entry + dex_gas_exit)
                     + (dex_protocol_fee_entry + dex_protocol_fee_exit)
                     + (slippage_entry + slippage_exit)
                     + bridge_fee  (if cross-chain)
                     + withdrawal_fee  (if moving between CEX/DEX)
```

**Worked Example (CEX-DEX arb, one direction):**

```
cex_fee (VIP1 taker)    = $35.00
dex_gas (Arbitrum)       =  $0.30
dex_protocol_fee         =  $0.00  (included in quote)
slippage (CEX)           = $15.00
withdrawal_fee (USDT)    =  $1.00
bridge_fee               =  $0.00  (same chain)

total_cost = 35.00 + 0.30 + 0.00 + 15.00 + 1.00 + 0.00
           = $51.30
```

### Net Profit

```
net_profit = gross_spread_usd - total_cost
```

```
gross_spread_usd = size_usd * spread_bps / 10000
```

**Worked Example:**

```
size_usd = 50,000
spread_bps = 21.4

gross_spread_usd = 50000 * 21.4 / 10000 = $107.00
total_cost = $51.30

net_profit = 107.00 - 51.30 = $55.70
```

### Profit-to-Cost Ratio

```
profit_to_cost = net_profit / total_cost
```

**Interpretation:**
- < 1.0x: Costs exceed profit -- do not trade
- 1.0x - 1.5x: Marginal -- high execution risk, consider skipping
- 1.5x - 3.0x: Acceptable -- proceed with caution
- 3.0x+: Attractive -- high confidence trade

### Minimum Spread for Breakeven

```
min_spread_bps = total_costs_usd / size_usd * 10000
```

### Funding Payment

```
funding_payment = size_usd * funding_rate * funding_intervals
```
- Positive funding_rate + short position = receive payment
- Positive funding_rate + long position = pay funding

### Borrow Cost

```
borrow_cost = size_usd * borrow_rate_annual * hold_days / 365
```

---

## 10. OKX CEX Trading Fee Tables

### Spot Trading Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.060% | 0.080% |
| VIP1 | >= 5M | 0.040% | 0.070% |
| VIP2 | >= 10M | 0.030% | 0.060% |
| VIP3 | >= 20M | 0.020% | 0.050% |
| VIP4 | >= 100M | 0.015% | 0.040% |
| VIP5 | >= 200M | 0.010% | 0.035% |

### USDT-Margined Swap (Perpetual) Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.045% |
| VIP2 | >= 10M | 0.010% | 0.040% |
| VIP3 | >= 20M | 0.008% | 0.035% |
| VIP4 | >= 100M | 0.005% | 0.030% |
| VIP5 | >= 200M | 0.002% | 0.025% |

### Coin-Margined Swap / Futures Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.045% |
| VIP2 | >= 10M | 0.010% | 0.040% |
| VIP3 | >= 20M | 0.008% | 0.035% |
| VIP4 | >= 100M | 0.005% | 0.030% |
| VIP5 | >= 200M | 0.002% | 0.025% |

### Fee Calculation Notes

- **Maker** = limit order that adds liquidity to the orderbook (not immediately matched)
- **Taker** = market order or limit order that immediately matches
- For **cost estimation**, assume taker fees unless the skill specifically uses limit orders
- Fees are deducted from the received asset (spot) or from margin (derivatives)

### Fee Formula

```
fee_usd = notional_size_usd * fee_rate

# Round-trip cost (open + close)
roundtrip_fee_usd = notional_size_usd * (entry_fee_rate + exit_fee_rate)
roundtrip_fee_bps = (entry_fee_rate + exit_fee_rate) * 10000
```

**Quick Reference: Round-Trip Taker Fees (bps):**

| Tier | Spot RT | Swap RT |
|------|---------|---------|
| VIP0 | 16.0 bps | 10.0 bps |
| VIP1 | 14.0 bps | 9.0 bps |
| VIP2 | 12.0 bps | 8.0 bps |
| VIP3 | 10.0 bps | 7.0 bps |
| VIP4 | 8.0 bps | 6.0 bps |
| VIP5 | 7.0 bps | 5.0 bps |

---

## 11. OKX Withdrawal Fee Tables

Withdrawal fees are flat (not percentage-based) and vary by asset and network.

### BTC Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Bitcoin (BTC) | 0.0001 BTC | 0.001 BTC |
| Lightning Network | 0.000001 BTC | 0.000001 BTC |

### ETH Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 0.00035 ETH | 0.001 ETH |
| Arbitrum One | 0.0001 ETH | 0.0001 ETH |
| Optimism | 0.00004 ETH | 0.0001 ETH |
| Base | 0.00004 ETH | 0.0001 ETH |
| zkSync Era | 0.000065 ETH | 0.0001 ETH |
| Linea | 0.00015 ETH | 0.0001 ETH |

### USDT Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDT | 10.0 USDT |
| Tron (TRC-20) | 1.0 USDT | 0.1 USDT |
| Polygon | 0.8 USDT | 0.1 USDT |
| Arbitrum One | 0.1 USDT | 0.1 USDT |
| Optimism | 0.1 USDT | 0.1 USDT |
| Base | 0.1 USDT | 0.1 USDT |
| Solana | 1.0 USDT | 1.0 USDT |
| BSC (BEP-20) | 0.3 USDT | 10.0 USDT |
| Avalanche C-Chain | 1.0 USDT | 1.0 USDT |
| TON | 0.5 USDT | 0.1 USDT |

### USDC Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDC | 10.0 USDC |
| Polygon | 0.8 USDC | 0.1 USDC |
| Arbitrum One | 0.1 USDC | 0.1 USDC |
| Optimism | 0.1 USDC | 0.1 USDC |
| Base | 0.1 USDC | 0.1 USDC |
| Solana | 1.0 USDC | 1.0 USDC |
| BSC (BEP-20) | 0.3 USDC | 10.0 USDC |
| Avalanche C-Chain | 1.0 USDC | 1.0 USDC |

### SOL Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Solana | 0.008 SOL | 0.1 SOL |

### Other Major Assets

| Asset | Network | Fee | Min Withdrawal |
|-------|---------|-----|---------------|
| BNB | BSC (BEP-20) | 0.0005 BNB | 0.01 BNB |
| MATIC | Polygon | 0.1 MATIC | 0.1 MATIC |
| AVAX | Avalanche C-Chain | 0.01 AVAX | 0.1 AVAX |
| ARB | Arbitrum One | 0.1 ARB | 1.0 ARB |
| OP | Optimism | 0.1 OP | 1.0 OP |
| OKB | X Layer | 0.001 OKB | 0.01 OKB |

### Withdrawal Fee Notes

- Withdrawal fees are **flat amounts**, not percentages.
- Fees are updated periodically based on network conditions.
- Always verify current fees via OKX API before calculating costs.
- Internal transfers between OKX accounts (sub-accounts) are **free**.
- Deposits to OKX are **free** (user only pays network gas).

---

## 12. Gas Benchmarks per Chain

Approximate gas costs for a typical DEX swap transaction.

| Chain | Avg Gas Price | Swap Gas Limit | Approx Cost (USD) | Native Token |
|-------|--------------|----------------|-------------------|-------------|
| Ethereum | 20-50 gwei | 150,000-300,000 | $5.00-$30.00 | ETH |
| Arbitrum | 0.1-0.5 gwei | 1,000,000-2,000,000 | $0.10-$0.50 | ETH |
| Base | 0.005-0.02 gwei | 150,000-300,000 | $0.01-$0.05 | ETH |
| Optimism | 0.005-0.02 gwei | 150,000-300,000 | $0.01-$0.05 | ETH |
| Polygon | 30-100 gwei | 150,000-300,000 | $0.01-$0.05 | MATIC |
| BSC | 3-5 gwei | 150,000-300,000 | $0.10-$0.30 | BNB |
| Avalanche | 25-30 nAVAX | 150,000-300,000 | $0.02-$0.10 | AVAX |
| Solana | N/A (lamports) | N/A (compute units) | $0.001-$0.01 | SOL |
| X Layer | 0.01-0.05 gwei | 150,000-300,000 | $0.001-$0.01 | OKB |

### Gas Notes

- **Ethereum L1** gas is by far the most expensive; avoid for small trades.
- **L2 costs** include a small L1 data posting fee that can spike during L1 congestion.
- **Multi-hop swaps** (going through multiple pools) use significantly more gas.
- **Approval transactions** (first-time token approvals) add ~46,000 gas on EVM chains.
- Gas prices fluctuate significantly; always fetch real-time estimates.

### Cost Efficiency by Chain (for $10,000 trade)

| Chain | Gas as % of Trade | Practical Minimum Trade |
|-------|-------------------|------------------------|
| Ethereum | 0.05-0.30% | $5,000+ |
| Arbitrum | 0.001-0.005% | $100+ |
| Base | 0.0001-0.0005% | $50+ |
| Solana | 0.00001-0.0001% | $10+ |
| BSC | 0.001-0.003% | $100+ |
| Polygon | 0.0001-0.0005% | $50+ |

### Solana Gas Details

- Base fee: 5,000 lamports (0.000005 SOL)
- Priority fee: 10,000-500,000 lamports depending on congestion
- Compute budget: 200,000-1,400,000 compute units

### Gas Price Fallback Chain

```
1. Try: onchainos onchain-gateway gas --chain {chain}
2. If fails: use benchmark midpoint from table above
   - Ethereum: 35 gwei
   - Arbitrum: 0.3 gwei
   - Base: 0.01 gwei
   - Solana: 55,000 lamports
3. Flag confidence as "medium" when using fallback
```

---

## 13. DEX Protocol Fee Tables

Fees charged by the DEX protocol itself (separate from gas). These are typically included in the quoted swap price from aggregators.

### Uniswap (Ethereum, Arbitrum, Base, Polygon, Optimism, BSC)

| Pool Tier | Fee | Typical Use |
|-----------|-----|------------|
| 0.01% | 1 bp | Stable-stable pairs (USDC/USDT) |
| 0.05% | 5 bps | Stable pairs, high-correlation pairs |
| 0.30% | 30 bps | Standard pairs (ETH/USDC, WBTC/ETH) |
| 1.00% | 100 bps | Exotic / low-liquidity pairs |

### Curve Finance (Ethereum, Arbitrum, Polygon, others)

| Pool Type | Fee |
|-----------|-----|
| Stablecoin pools | 0.01-0.04% |
| Crypto pools (v2) | 0.04-0.40% |
| Factory pools | Variable |

### PancakeSwap (BSC, Ethereum, Arbitrum, Base)

| Pool Tier | Fee |
|-----------|-----|
| V3 (stable) | 0.01% |
| V3 (standard) | 0.25% |
| V3 (exotic) | 1.00% |
| V2 | 0.25% |

### Jupiter (Solana)

- Jupiter itself charges **no protocol fee** (aggregator layer)
- Underlying pool fees vary:
  - Orca Whirlpools: 0.01%-2.00% (tier-based)
  - Raydium: 0.25% (standard), 0.01% (concentrated)
  - Meteora: 0.01%-1.00% (dynamic)
- Route optimization selects the best fee-adjusted path automatically

### DEX Fee Notes

- **Aggregator quotes already include DEX fees** in the output amount. Do not double-count.
- When comparing CEX vs DEX prices, the DEX price is already net of protocol fees.
- The OKX DEX API aggregates across multiple DEX protocols and returns the best net quote.
- Some protocols have **dynamic fees** that adjust based on volatility.

---

## 14. Bridge Fee Tables

Typical fees for cross-chain asset transfers. Highly variable by bridge, route, and conditions.

### Major Bridge Protocols

| Bridge | Typical Fee Range | Speed | Notes |
|--------|------------------|-------|-------|
| OKX Bridge (via DEX API) | 0-0.1% | 1-15 min | Aggregates multiple bridges |
| Across Protocol | 0.04-0.12% | 1-5 min | Fast, uses relayers |
| Stargate | 0.06% | 5-15 min | LayerZero-based |
| Hop Protocol | 0.05-0.20% | 2-10 min | Uses AMMs |
| Synapse | 0.05-0.15% | 5-15 min | Multi-chain AMM |
| Native Bridges | Free (gas only) | 7 min - 7 days | Slowest, cheapest |

### Native Bridge Wait Times

| Route | Deposit Time | Withdrawal Time |
|-------|-------------|-----------------|
| Ethereum -> Arbitrum | ~10 min | ~7 days (challenge period) |
| Ethereum -> Optimism | ~10 min | ~7 days (challenge period) |
| Ethereum -> Base | ~10 min | ~7 days (challenge period) |
| Ethereum -> Polygon | ~10 min | ~30 min (PoS bridge) |
| Ethereum -> BSC | N/A | N/A (use 3rd party) |

### Bridge Fee Notes

- **OKX withdrawal is often cheaper than bridging** for moving assets from CEX to another chain. Withdraw directly to the target chain when supported.
- Bridge fees include: protocol fee + gas on source chain + gas on destination chain.
- For large amounts (>$100K), verify bridge liquidity on the destination chain to avoid high slippage.
- When possible, avoid bridging altogether by using native CEX withdrawal to the target chain.

### CEX Withdrawal vs Bridge Comparison

When moving USDT from Ethereum to Arbitrum:

| Method | Cost | Time |
|--------|------|------|
| OKX withdraw to Arbitrum directly | 0.1 USDT | 1-5 min |
| Bridge via Across | ~$2-5 (0.04-0.1%) + gas | 1-5 min |
| Bridge via native bridge | Gas only (~$15-30) | ~10 min + 7 days |

**Recommendation:** Always prefer direct CEX withdrawal to target chain when possible. It is almost always the cheapest and fastest route.

---

## 15. Cost Summary for Common Trade Patterns

### Pattern A: CEX-DEX Arbitrage (Buy CEX, Sell DEX on Arbitrum)

```
Trade size: $50,000

CEX buy (VIP1 taker):         $35.00  (7.0 bps)
CEX withdrawal (ETH to Arb):   $0.34  (0.0001 ETH)
DEX sell gas (Arbitrum):        $0.30
DEX slippage:                  $10.00  (est. 2 bps)
                               ------
Total one-way cost:            $45.64  (9.1 bps)
```

### Pattern B: Funding Harvest (Spot long + Perp short)

```
Position size: $100,000

Spot buy (VIP1 taker):        $70.00  (7.0 bps)
Perp short (VIP1 taker):      $45.00  (4.5 bps)
                               ------
Entry cost:                   $115.00  (11.5 bps)
Exit cost (same):             $115.00  (11.5 bps)
Round-trip:                   $230.00  (23.0 bps)
```

### Pattern C: Basis Trade (Spot long + Futures short)

```
Position size: $100,000

Spot buy (VIP1 taker):        $70.00  (7.0 bps)
Futures short (VIP1 taker):   $45.00  (4.5 bps)
                               ------
Entry cost:                   $115.00  (11.5 bps)
Exit at expiry: futures settle automatically, close spot:
  Spot sell:                   $70.00  (7.0 bps)
                               ------
Total cost:                   $185.00  (18.5 bps)
```

---

## 16. Supported Chains Table

| Chain | chainIndex (OKX) | Native Token | Native Token Address (DEX) |
|-------|------------------|-------------|---------------------------|
| Ethereum | 1 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| BSC | 56 | BNB | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| Polygon | 137 | MATIC | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| Arbitrum One | 42161 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| Optimism | 10 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| Base | 8453 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| Avalanche C-Chain | 43114 | AVAX | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |
| Solana | 501 | SOL | `11111111111111111111111111111111` |
| X Layer | 196 | OKB | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` |

### Critical Warnings

**Solana Native Token:**
- `11111111111111111111111111111111` = native SOL (System Program ID) -- use this
- `So11111111111111111111111111111111111111112` = Wrapped SOL (wSOL) -- do NOT use for native

**EVM Native Token:** All EVM chains use `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` as the placeholder for native tokens.

**Address Formatting:**
- EVM: lowercase required
- Solana: base58, case-sensitive

**BSC USDT Warning:** BSC USDT has **18 decimals**, unlike most other chains where it has 6 decimals. Always check the token's `decimals` value before encoding amounts.

---

## 17. Safety Checks

### Profitability Safety Checks (#10-11 from Pre-Trade Checklist)

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 10 | Gas vs profit ratio | `onchain-gateway gas` + calculation | Gas cost > 30% of gross spread | Gas cost > 10% of spread | `NOT_PROFITABLE` |
| 11 | Net profitability | `profitability-calculator` | `net_profit <= 0` | `profit_to_cost_ratio < 2` | `NOT_PROFITABLE` |

### Full Pre-Trade Safety Checklist (context)

Every skill runs through these checks **in order** before producing a recommendation:

| # | Check | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|----------------|----------------|------------|
| 1 | MCP connectivity | Server not reachable | -- | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `authenticated: false` | -- | `AUTH_FAILED` |
| 3 | Data freshness | > 60s stale (> 5s for arb) | > 30s stale | `DATA_STALE` |
| 4 | Token honeypot | `is_honeypot === "1"` | -- | `SECURITY_BLOCKED` |
| 5 | Token tax rate | buy > 5% OR sell > 10% | buy > 1% | `SECURITY_BLOCKED` |
| 6 | Holder concentration | Top 10 non-contract > 80% | > 50% | `SECURITY_BLOCKED` |
| 7 | Contract verified | `is_open_source === "0"` | -- | `SECURITY_BLOCKED` |
| 8 | Liquidity depth | `liquidityUsd < $100,000` | `< $500,000` | `INSUFFICIENT_LIQUIDITY` |
| 9 | Price impact | `priceImpactPercent > 2%` | `> 0.5%` | `INSUFFICIENT_LIQUIDITY` |
| 10 | Gas vs profit ratio | Gas > 30% of spread | Gas > 10% of spread | `NOT_PROFITABLE` |
| 11 | Net profitability | `net_profit <= 0` | `profit_to_cost_ratio < 2` | `NOT_PROFITABLE` |

### BLOCK Conditions

| Condition | Action | Error Code |
|-----------|--------|------------|
| `net_profit <= 0` | Output `[NOT PROFITABLE]` with full breakdown. Do not recommend. | `NOT_PROFITABLE` |
| `total_costs_usd` cannot be computed (missing data) | Output error with details. | `FEE_LOOKUP_FAILED` |
| Required field missing in TradeLeg | Output validation error. | `MISSING_FIELD` |
| Invalid field value | Output validation error. | `INVALID_LEG` |

### WARN Conditions

| Condition | Warning Message |
|-----------|----------------|
| `profit_to_cost_ratio < 2.0` | "利潤率偏低，執行風險大 / Thin margin, execution risk is high" |
| `gas_cost > 30% of gross_spread` | "Gas 費佔毛利 {pct}%，考慮使用 L2 / Gas is {pct}% of gross, consider L2" |
| `estimated_slippage > 100 bps (1%)` | "滑點估計偏高 ({bps} bps)，減小規模或分批執行 / High slippage, reduce size or split" |
| `withdrawal_fee > 10% of net_profit` | "提幣費佔淨利 {pct}%，考慮直接提到目標鏈 / Withdrawal fee is {pct}% of profit" |
| `confidence == "low"` | "數據來源不完整，結果僅供參考 / Incomplete data sources, treat as estimate only" |
| `hold_days > 7 and funding_rate used` | "長期持有假設固定資金費率，實際會波動 / Long hold assumes constant funding, actual varies" |

---

## 18. Error Codes

| Code | Condition | User Message (ZH) | User Message (EN) |
|------|-----------|-------------------|-------------------|
| `INVALID_LEG` | Leg field has invalid value | 交易腿參數無效：{field} = {value} | Invalid trade leg parameter: {field} = {value} |
| `MISSING_FIELD` | Required field missing | 缺少必要參數：{field}（{leg_description}） | Missing required field: {field} ({leg_description}) |
| `FEE_LOOKUP_FAILED` | Cannot determine fee rate | 費率查詢失敗：{detail}。請確認 VIP 等級和交易類型。 | Fee lookup failed: {detail}. Verify VIP tier and instrument type. |
| `GAS_FETCH_FAILED` | Cannot get live gas price | Gas 價格獲取失敗，使用基準值估算（信心度降為 medium） | Gas price fetch failed. Using benchmark estimate (confidence: medium). |
| `ORDERBOOK_EMPTY` | Orderbook returned no levels | 訂單簿為空或流動性不足：{instId} | Orderbook empty or insufficient liquidity: {instId} |
| `NOT_PROFITABLE` | Net profit <= 0 | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行 | Net profit is negative after all costs ({net_pnl}). Not recommended. |
| `MARGIN_TOO_THIN` | Profit-to-cost < 2.0 | 利潤空間偏薄（利潤/成本比 = {ratio}x），風險較高 | Thin margin (profit/cost = {ratio}x). Higher execution risk. |
| `MCP_NOT_CONNECTED` | MCP server unreachable | MCP 伺服器無法連線，請檢查 okx-trade-mcp 是否啟動 | MCP server unreachable. Check if okx-trade-mcp is running. |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗，請檢查 OKX API 金鑰設定 | API authentication failed. Check OKX API key configuration. |
| `DATA_STALE` | Price/market data too old | 市場數據已過期（超過 {threshold} 秒），正在重新獲取... | Market data stale (> {threshold}s). Refetching... |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. |

---

## 19. Cross-Skill Integration

### Input Contract: Who Calls This Skill

| Calling Skill | When | Data Provided |
|--------------|------|---------------|
| `cex-dex-arbitrage` | After finding spread opportunity | TradeLeg[] with CEX + DEX legs, prices pre-filled |
| `funding-rate-arbitrage` | After finding funding opportunity | TradeLeg[] with spot + swap legs, funding_rate + intervals |
| `basis-trading` | After finding basis opportunity | TradeLeg[] with spot + futures legs, hold_days |
| `yield-optimizer` | When computing switching costs | TradeLeg[] representing the rebalance (exit old + enter new) |
| `smart-money-tracker` | Before recommending copy trade | TradeLeg[] for the proposed copy |
| User (direct) | Interactive cost analysis | TradeLeg[] manually constructed |

### Output Contract: What This Skill Returns

All callers receive a `ProfitabilityResult` (see Section 7.1).

| Consuming Skill | Fields Used | Decision Logic |
|----------------|-------------|---------------|
| `cex-dex-arbitrage` | `is_profitable`, `net_profit_usd`, `profit_to_cost_ratio` | Skip if `is_profitable == false` or `profit_to_cost_ratio < 1.5` |
| `funding-rate-arbitrage` | `total_costs_usd` (as entry cost), `min_spread_for_breakeven_bps` | Compute break-even holding days: `total_costs / daily_income` |
| `basis-trading` | `total_costs_usd`, `cost_breakdown` | Subtract from basis yield to get net yield |
| `yield-optimizer` | `total_costs_usd` (switching cost) | Compute break-even period for yield switch |
| `smart-money-tracker` | `is_profitable`, `warnings` | Gate copy recommendation |

### Data Flow Diagram

```
price-feed-aggregator.snapshot
  │
  │  PriceSnapshot[] (CEX + DEX prices)
  ▼
[strategy skill].scan / evaluate
  │
  │  Constructs TradeLeg[] with prices
  ▼
profitability-calculator.estimate
  │
  │  ProfitabilityResult
  ▼
[strategy skill] applies go/no-go decision
  │
  ▼
Output to user (with cost breakdown)
```

---

## 20. Output Templates

### 20.1 Estimate / Breakdown Output

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

## 盈利性分析 / Profitability Analysis

策略: CEX 買入 -> DEX 賣出 (ETH on Ethereum)
規模: $1,000.00
VIP 等級: VIP0

── 成本分解 ──────────────────────────────

| 項目 | 金額 | 佔毛利 |
|------|------|--------|
| 毛利差 (Gross Spread) | +$17.77 | 100.0% |
| CEX Taker 費 (0.08%) | -$0.80 | 4.5% |
| DEX Gas 費 (Ethereum) | -$5.20 | 29.3% |
| DEX 滑點 (est. 12 bps) | -$1.20 | 6.8% |
| OKX 提幣費 (ETH ERC-20) | -$1.55 | 8.7% |
| **總成本** | **-$8.75** | **49.2%** |
| **淨利潤** | **+$9.02** | **50.8%** |

利潤/成本比: 1.03x
最小盈利價差: 88 bps
信心度: HIGH

結果: [PROFITABLE]

  注意:
  - Gas 費佔毛利 29.3%，接近 30% 閾值
    考慮使用 Layer 2 (Arbitrum/Base) 降低 Gas 成本
  - 利潤/成本比 1.03x 低於 2.0x 建議閾值
    執行風險較大，需精確時機

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 考慮切換至 Arbitrum 降低 Gas 成本:
     profitability-calculator estimate --legs '[...chain:arbitrum...]'
  2. 查看不同規模的敏感度分析:
     profitability-calculator sensitivity --legs '[...]' --variable size_usd --range '[500,5000]'
  3. 查看最低價差要求:
     profitability-calculator min-spread --venue-a okx-cex --venue-b okx-dex --chain ethereum --asset ETH --size-usd 1000

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

### 20.2 Min-Spread Output

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

## 最小價差分析 / Minimum Spread Analysis

路線: OKX (CEX) <-> OKX DEX (Ethereum)
資產: ETH | 規模: $10,000.00 | VIP: VIP0

── 成本組成 (bps) ────────────────────────

| 成本項目 | bps |
|----------|-----|
| CEX Taker 費 | 8.0 |
| DEX Gas 費 | 1.6 |
| 滑點估計 | 3.0 |
| 提幣費 | 1.2 |
| **合計 (最小價差)** | **13.8** |

結論: 至少需要 14 bps 的價差才能盈虧平衡 ($10,000 規模)。

══════════════════════════════════════════
```

### 20.3 Sensitivity Output

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

## 敏感度分析 / Sensitivity Analysis

變量: size_usd (交易規模)
基礎策略: CEX 買入 -> DEX 賣出 (ETH, Arbitrum)

| 規模 | 總成本 | 淨利潤 | 利潤/成本 | 盈利? |
|------|--------|--------|-----------|-------|
| $1,000 | $1.55 | +$0.59 | 0.38x | [WARN] |
| $3,000 | $2.95 | +$3.47 | 1.18x | [OK] |
| $5,000 | $4.35 | +$6.35 | 1.46x | [OK] |
| $10,000 | $7.50 | +$14.20 | 1.89x | [OK] |
| $20,000 | $13.80 | +$29.60 | 2.14x | [GOOD] |
| $50,000 | $33.00 | +$75.70 | 2.29x | [GOOD] |

盈虧平衡點: ~$620
最佳利潤/成本比: $50,000 (2.29x)

結論: 規模越大，固定成本（Gas、提幣費）攤薄越多。
建議最小規模: $3,000+（利潤/成本比 > 1.0x）

══════════════════════════════════════════
```

---

## 21. Conversation Examples

### Example 1: Direct Profitability Check (CEX-DEX Arbitrage)

**User:** ETH OKX 現貨 3412.50，Uniswap 上 3419.80，$5000 規模，賺唔賺？

**Assistant Flow:**

```
1. Parse: Buy OKX CEX at 3412.50, Sell Uniswap (Ethereum) at 3419.80, $5,000

2. Construct TradeLeg[]:
   Leg 1: {venue: "okx-cex", asset: "ETH", side: "buy", size_usd: 5000, order_type: "market"}
   Leg 2: {venue: "uniswap", chain: "ethereum", asset: "ETH", side: "sell", size_usd: 5000,
           withdrawal_required: true}

3. Fetch live data:
   - market_get_orderbook(instId="ETH-USDT", sz="400")
     → Walk asks to estimate slippage for $5,000 buy
     → Slippage: ~2 bps
   - onchainos onchain-gateway gas --chain ethereum
     → gasPriceGwei: "28"
   - Gas limit benchmark: 200,000 (Uniswap swap)
   - Native token price: $3,412.50 (from ticker)
   - Withdrawal fee: 0.00035 ETH = $1.19

4. Compute:
   gross_spread = (3419.80 - 3412.50) / 3412.50 * 5000 = $10.69

   cex_trading_fee = 5000 * 0.0008 = $4.00
   gas_cost = 28 * 200000 * 3412.50 / 1e9 = $19.11
   slippage_cost = 5000 * 2 / 10000 = $1.00
   withdrawal_fee = 0.00035 * 3412.50 = $1.19

   total_costs = 4.00 + 19.11 + 1.00 + 1.19 = $25.30
   net_profit = 10.69 - 25.30 = -$14.61

5. Result: NOT PROFITABLE
```

**Output:**

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

## 盈利性分析

策略: OKX 買入 -> Uniswap 賣出 (ETH on Ethereum)
規模: $5,000.00

── 成本分解 ──────────────────────────────

| 項目 | 金額 | 佔毛利 |
|------|------|--------|
| 毛利差 | +$10.69 | 100.0% |
| CEX Taker 費 (0.08%) | -$4.00 | 37.4% |
| DEX Gas 費 (28 gwei) | -$19.11 | 178.8% |
| CEX 滑點 (2 bps) | -$1.00 | 9.4% |
| 提幣費 (ETH ERC-20) | -$1.19 | 11.1% |
| **總成本** | **-$25.30** | **236.7%** |
| **淨利潤** | **-$14.61** | -- |

結果: [NOT PROFITABLE]

  主要原因: Ethereum Gas 費 ($19.11) 遠超毛利 ($10.69)。

  建議:
  1. 改用 Arbitrum: Gas 約 $0.30，預估淨利 +$4.20
  2. 增大規模至 $20,000+: 攤薄固定成本
  3. 等待 Gas 降至 <10 gwei 再行動

══════════════════════════════════════════
```

---

### Example 2: Funding Rate Carry Trade Cost Analysis

**User:** I want to do ETH funding harvest, $50,000 position, VIP1. What's the cost to enter and exit?

**Assistant Flow:**

```
1. Construct TradeLeg[] for entry:
   Leg 1: {venue: "okx-cex", asset: "ETH", side: "buy", size_usd: 50000,
           instrument_type: "spot", vip_tier: "VIP1"}
   Leg 2: {venue: "okx-cex", asset: "ETH", side: "sell", size_usd: 50000,
           instrument_type: "swap", vip_tier: "VIP1"}

2. Compute entry cost:
   spot_buy_fee = 50000 * 0.0007 = $35.00 (VIP1 spot taker)
   swap_sell_fee = 50000 * 0.00045 = $22.50 (VIP1 swap taker)
   slippage (spot): fetch orderbook → ~1 bps → $5.00
   slippage (swap): fetch orderbook → ~1 bps → $5.00

   entry_cost = 35.00 + 22.50 + 5.00 + 5.00 = $67.50

3. Exit cost is symmetric:
   exit_cost = $67.50

4. Round-trip total: $135.00

5. Fetch current funding rate:
   market_get_funding_rate(instId="ETH-USDT-SWAP")
   → fundingRate: "0.000150" (0.015% per 8h)

   daily_income = 50000 * 0.000150 * 3 = $22.50
   break_even_days = 135.00 / 22.50 = 6.0 days
```

**Output:**

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

## 成本分析: ETH 資金費率套利

規模: $50,000.00 | VIP: VIP1

── 建倉成本 ──────────────────────────────

| 項目 | 金額 | bps |
|------|------|-----|
| 現貨買入手續費 (VIP1 taker) | $35.00 | 7.0 |
| 永續合約做空手續費 (VIP1 taker) | $22.50 | 4.5 |
| 現貨滑點 (est. 1 bps) | $5.00 | 1.0 |
| 永續滑點 (est. 1 bps) | $5.00 | 1.0 |
| **建倉總成本** | **$67.50** | **13.5** |

── 平倉成本 (對稱) ───────────────────────

| 項目 | 金額 | bps |
|------|------|-----|
| **平倉總成本** | **$67.50** | **13.5** |

── 總結 ──────────────────────────────────

| 指標 | 數值 |
|------|------|
| 總進出成本 | $135.00 (27.0 bps) |
| 當前資金費率 | +0.015% / 8h (16.43% ann.) |
| 每日預估收入 | +$22.50 |
| 盈虧平衡天數 | 6.0 天 |

結果: [PROFITABLE] (持有 > 6 天後)

  注意: 資金費率每 8 小時變動一次，以上預估假設費率不變。
  建議持續監控費率走勢。

══════════════════════════════════════════
```

---

### Example 3: Min-Spread with Chain Comparison

**User:** ETH 在 OKX 和 DEX 之間最少要幾 bps 價差先有得賺？$10,000，比較 Ethereum 同 Arbitrum。

**Assistant Flow:**

```
1. Run min-spread for Ethereum:
   profitability-calculator min-spread --venue-a okx-cex --venue-b okx-dex
     --chain ethereum --asset ETH --size-usd 10000

   CEX taker (VIP0): 8.0 bps → $8.00
   Gas (Ethereum, 30 gwei, 200k limit): $20.70 → 20.7 bps
   Slippage: ~2 bps → $2.00
   Withdrawal (ETH ERC-20): $1.19 → 1.2 bps
   Total: 31.9 bps

2. Run min-spread for Arbitrum:
   profitability-calculator min-spread --venue-a okx-cex --venue-b okx-dex
     --chain arbitrum --asset ETH --size-usd 10000

   CEX taker (VIP0): 8.0 bps → $8.00
   Gas (Arbitrum, 0.3 gwei, 1.5M limit): $1.55 → 1.6 bps
   Slippage: ~2 bps → $2.00
   Withdrawal (ETH Arbitrum): $0.35 → 0.4 bps
   Total: 12.0 bps
```

**Output:**

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

## 最小價差比較: Ethereum vs Arbitrum

資產: ETH | 規模: $10,000.00 | VIP: VIP0

── Ethereum ──────────────────────────────

| 成本項目 | bps | USD |
|----------|-----|-----|
| CEX Taker 費 | 8.0 | $8.00 |
| Gas 費 (30 gwei) | 20.7 | $20.70 |
| 滑點估計 | 2.0 | $2.00 |
| 提幣費 (ERC-20) | 1.2 | $1.19 |
| **合計** | **31.9** | **$31.89** |

── Arbitrum ──────────────────────────────

| 成本項目 | bps | USD |
|----------|-----|-----|
| CEX Taker 費 | 8.0 | $8.00 |
| Gas 費 (0.3 gwei) | 1.6 | $1.55 |
| 滑點估計 | 2.0 | $2.00 |
| 提幣費 (Arbitrum) | 0.4 | $0.35 |
| **合計** | **12.0** | **$11.90** |

── 比較 ──────────────────────────────────

| 鏈 | 最小價差 | 節省 |
|----|---------|------|
| Ethereum | 32 bps | -- |
| Arbitrum | 12 bps | -20 bps (62%) |

結論: Arbitrum 所需最小價差僅為 Ethereum 的 38%。
Gas 費是主要差異來源（20.7 bps vs 1.6 bps）。

強烈建議使用 Arbitrum 執行 ETH CEX-DEX 套利。

══════════════════════════════════════════
```

---

## 22. Implementation Notes

### Orderbook Depth Parsing

When auto-calculating slippage from `market_get_orderbook`:

```
1. Call: market_get_orderbook(instId="ETH-USDT", sz="400")
2. Parse bids (for sell) or asks (for buy) - remember all values are STRINGS
3. Convert size_usd to asset quantity: qty = size_usd / best_price
4. Walk levels:
   remaining = qty
   weighted_sum = 0
   for level in levels:
     level_price = parseFloat(level[0])
     level_qty = parseFloat(level[1])
     fill = min(remaining, level_qty)
     weighted_sum += level_price * fill
     remaining -= fill
     if remaining <= 0: break
5. vwap = weighted_sum / qty
6. impact_bps = abs(vwap - best_price) / best_price * 10000
```

### Fee Rate Lookup Priority

```
1. If user provides explicit fee_rate → use it
2. If vip_tier + instrument_type provided → lookup from fee tables above
3. Default: VIP0 taker rate for the instrument type
   - Spot: 0.080%
   - Swap/Futures: 0.050%
```

---

## 23. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 4 commands: estimate, breakdown, min-spread, sensitivity. |
| 2026-03-09 | 1.1.0 | Inlined all reference file content for OpenClaw/Lark compatibility. |

---

**Note:** All content from reference files has been inlined above for OpenClaw/Lark compatibility. The files below exist for human maintenance only.

| File | Relevance to This Skill |
|------|------------------------|
| `references/formulas.md` | Calculation formulas (Sections 1-2, 7) |
| `references/fee-schedule.md` | OKX fee tiers, withdrawal fees, gas benchmarks, DEX fees, bridge fees |
| `references/chain-reference.md` | Supported chains, native addresses, chainIndex mapping |
| `references/safety-checks.md` | Pre-trade safety checklist, profitability checks #10-11 |
