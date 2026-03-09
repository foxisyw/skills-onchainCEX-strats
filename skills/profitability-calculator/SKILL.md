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
- **Check token security** -- delegate to GoPlus MCP (see `references/goplus-tools.md`)
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
| `references/fee-schedule.md` | Static fee tables, withdrawal fee lookup | Yes (built-in) |
| `references/formulas.md` | Calculation formulas | Yes (built-in) |

### Pre-flight Checks

```
1. Call system_get_capabilities
   - Verify authenticated: true
   - Note mode: "demo" or "live"

2. If trade has onchain legs:
   - Verify OnchainOS CLI is available: `which onchainos`
   - If unavailable: use benchmark gas values from fee-schedule.md
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
| `chain` | string | Required when venue is not `okx-cex`. Must match `references/chain-reference.md` | `MISSING_FIELD` |
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
    withdrawal_fee: number       # Fixed per-asset from OKX schedule (fee-schedule.md)
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
| `medium` | Some data from benchmarks (e.g. gas from fee-schedule.md, not live). |
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
  data_source: string    # Where the input came from ("live orderbook", "fee-schedule.md", "user input")
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
    ├─ Fee rate: lookup from references/fee-schedule.md using vip_tier + instrument_type
    │   - Spot VIP0 taker: 0.080%, maker: 0.060%
    │   - Swap VIP0 taker: 0.050%, maker: 0.020%
    │
    ├─ Slippage: if not provided, fetch orderbook:
    │   market_get_orderbook(instId={ASSET}-USDT, sz="400")
    │   Walk bids/asks to compute volume-weighted avg price at size_usd
    │   price_impact_bps = (vwap - best_price) / best_price * 10000
    │
    └─ Withdrawal fee: if withdrawal_required=true
        Lookup from fee-schedule.md → fixed amount per asset+network

  IF leg.venue is DEX (okx-dex, uniswap, jupiter, etc.):
    ├─ Gas price: if gas_gwei not provided:
    │   onchainos onchain-gateway gas --chain {chain}
    │   On failure → use benchmark from fee-schedule.md, set confidence="medium"
    │
    ├─ Gas limit: use benchmarks from fee-schedule.md:
    │   - Ethereum: 150,000-300,000
    │   - Arbitrum: 1,000,000-2,000,000
    │   - Solana: N/A (use lamport-based calculation)
    │   Or fetch via onchainos onchain-gateway gas-limit if calldata available
    │
    ├─ DEX protocol fee: lookup from fee-schedule.md:
    │   - Uniswap standard: 30 bps
    │   - Jupiter: 0 bps (aggregator layer)
    │   - Note: DEX quotes from aggregators already include protocol fees
    │     so set dex_trading_fee = 0 when using aggregator quotes
    │
    └─ Slippage: if not provided:
        Use price impact from dex-swap quote if available
        Otherwise estimate from fee-schedule.md benchmarks

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
# Formulas (cross-reference: references/formulas.md)

cex_trading_fee = size_usd * fee_rate
  - fee_rate: 0.0008 (taker VIP0 spot) or 0.0006 (maker VIP0 spot)
  - For swaps: 0.0005 (taker VIP0) or 0.0002 (maker VIP0)

gas_cost = gas_gwei * gas_limit * native_token_price_usd / 1e9
  - Example: 30 gwei * 150,000 * $3,450 / 1e9 = $15.53
  - Solana: (base_fee_lamports + priority_fee_lamports) * sol_price / 1e9

slippage_cost = size_usd * estimated_slippage_bps / 10000
  - Auto-calc: walk orderbook to find VWAP at size_usd
  - price_impact_bps = (vwap - best_price) / best_price * 10000

withdrawal_fee = lookup from fee-schedule.md
  - ETH (ERC-20): 0.00035 ETH (~$1.21 at $3,450)
  - ETH (Arbitrum): 0.0001 ETH (~$0.35)
  - USDT (TRC-20): 1.0 USDT
  - USDT (Arbitrum): 0.1 USDT

bridge_fee = estimated from fee-schedule.md Section 5
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

> All formulas here are canonical. For full derivations and worked examples, see
> `references/formulas.md` Sections 1-2 and 7.

### CEX Trading Fee

```
cex_fee = size_usd * fee_rate

Quick Reference (Spot Taker):
  VIP0: 0.080% = 8.0 bps
  VIP1: 0.070% = 7.0 bps
  VIP2: 0.060% = 6.0 bps
  VIP3: 0.050% = 5.0 bps

Quick Reference (Swap Taker):
  VIP0: 0.050% = 5.0 bps
  VIP1: 0.045% = 4.5 bps
  VIP2: 0.040% = 4.0 bps
```

### Gas Cost (EVM)

```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price_usd / 1e9

Example (Ethereum):
  30 gwei * 150,000 * $3,450 / 1e9 = $15.53

Example (Arbitrum):
  0.3 gwei * 1,500,000 * $3,450 / 1e9 = $1.55

Example (Base):
  0.01 gwei * 200,000 * $3,450 / 1e9 = $0.007
```

### Gas Cost (Solana)

```
gas_cost_usd = (base_fee_lamports + priority_fee_lamports) * sol_price_usd / 1e9

Example:
  (5,000 + 50,000) * $145 / 1e9 = $0.008
```

### Slippage Cost

```
slippage_cost = size_usd * estimated_slippage_bps / 10000

Auto-calculation from orderbook:
  1. Fetch orderbook: market_get_orderbook(instId, sz="400")
  2. Walk levels until cumulative size >= order_size
  3. vwap = sum(price_i * qty_i) / sum(qty_i)
  4. price_impact_bps = abs(vwap - best_price) / best_price * 10000
  5. slippage_cost = size_usd * price_impact_bps / 10000
```

### Withdrawal Fee

```
Flat fee per asset + network. Lookup from references/fee-schedule.md.

Common values:
  ETH (ERC-20):    0.00035 ETH
  ETH (Arbitrum):  0.0001 ETH
  ETH (Base):      0.00004 ETH
  USDT (TRC-20):   1.0 USDT
  USDT (Arbitrum):  0.1 USDT
  USDT (ERC-20):   3.0 USDT
  SOL (Solana):    0.008 SOL

Convert to USD: withdrawal_fee_usd = fee_amount * asset_price_usd
```

### Net Profit

```
net_profit = gross_spread_usd - total_costs_usd
profit_to_cost_ratio = net_profit / total_costs_usd
min_spread_bps = total_costs_usd / size_usd * 10000
```

---

## 10. Safety Checks

> Cross-reference: `references/safety-checks.md` Section 1, checks #10-11.

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

## 11. Error Codes

| Code | Condition | User Message (ZH) | User Message (EN) |
|------|-----------|-------------------|-------------------|
| `INVALID_LEG` | Leg field has invalid value | 交易腿參數無效：{field} = {value} | Invalid trade leg parameter: {field} = {value} |
| `MISSING_FIELD` | Required field missing | 缺少必要參數：{field}（{leg_description}） | Missing required field: {field} ({leg_description}) |
| `FEE_LOOKUP_FAILED` | Cannot determine fee rate | 費率查詢失敗：{detail}。請確認 VIP 等級和交易類型。 | Fee lookup failed: {detail}. Verify VIP tier and instrument type. |
| `GAS_FETCH_FAILED` | Cannot get live gas price | Gas 價格獲取失敗，使用基準值估算（信心度降為 medium） | Gas price fetch failed. Using benchmark estimate (confidence: medium). |
| `ORDERBOOK_EMPTY` | Orderbook returned no levels | 訂單簿為空或流動性不足：{instId} | Orderbook empty or insufficient liquidity: {instId} |
| `NOT_PROFITABLE` | Net profit <= 0 | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行 | Net profit is negative after all costs ({net_pnl}). Not recommended. |
| `MARGIN_TOO_THIN` | Profit-to-cost < 2.0 | 利潤空間偏薄（利潤/成本比 = {ratio}x），風險較高 | Thin margin (profit/cost = {ratio}x). Higher execution risk. |

---

## 12. Cross-Skill Integration

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

## 13. Output Templates

### 13.1 Estimate / Breakdown Output

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

### 13.2 Min-Spread Output

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

### 13.3 Sensitivity Output

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

## 14. Conversation Examples

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
   - Withdrawal fee: 0.00035 ETH = $1.19 (fee-schedule.md)

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

## 15. Implementation Notes

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
2. If vip_tier + instrument_type provided → lookup from fee-schedule.md
3. Default: VIP0 taker rate for the instrument type
   - Spot: 0.080%
   - Swap/Futures: 0.050%
```

### Gas Price Fallback Chain

```
1. Try: onchainos onchain-gateway gas --chain {chain}
2. If fails: use benchmark midpoint from fee-schedule.md Section 3
   - Ethereum: 35 gwei
   - Arbitrum: 0.3 gwei
   - Base: 0.01 gwei
   - Solana: 55,000 lamports
3. Flag confidence as "medium" when using fallback
```

---

## 16. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 4 commands: estimate, breakdown, min-spread, sensitivity. |
