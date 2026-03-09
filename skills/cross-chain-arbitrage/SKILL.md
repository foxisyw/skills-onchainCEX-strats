---
name: cross-chain-arbitrage
description: >
  Multi-chain price differential arbitrage with CEX perpetual hedge during bridge transit.
  Buy on cheap chain, bridge to expensive chain, sell — while hedging price risk via OKX
  short during the 1-5 minute bridge window. Requires $50K+ capital for profitability due
  to fixed costs (gas, bridge fees, slippage on both chains, hedge open/close).
  Trigger phrases include: "cross-chain", "跨鏈套利", "bridge arbitrage", "跨鏈價差",
  "bridge arb", "鏈間套利", "multi-chain arb", "跨鏈搬磚", "chain arbitrage",
  "L2 arbitrage", "cross chain spread", "橋接套利", "Across", "Stargate",
  "bridge and sell", "transit hedge", "橋接對沖".
  Do NOT use for: single-chain CEX-DEX arbitrage (use cex-dex-arbitrage),
  funding rate carry (use funding-rate-arbitrage), basis trading (use basis-trading),
  yield comparison (use yield-optimizer), stablecoin depeg (use stablecoin-depeg-arbitrage),
  trade execution (no skill executes trades).
  Requires: okx-trade-mcp (CEX data + hedge quotes) + OnchainOS CLI (DEX data on both chains)
  + GoPlus MCP (token security on both chains).
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

# cross-chain-arbitrage

Multi-chain price differential arbitrage skill for the Onchain x CEX Strats system. Detects price discrepancies for the same asset across different blockchains, evaluates bridge costs and transit times, and recommends hedged cross-chain arbitrage trades. The core strategy: **buy on the cheaper chain's DEX, bridge to the expensive chain, sell on the expensive chain's DEX -- while holding a matching OKX perpetual SHORT as a hedge during bridge transit** to eliminate price risk during the 1-5 minute bridge window.

Key constraint: Fixed costs (gas on source + destination, bridge fee, slippage on both DEX swaps, hedge open/close fees) require **minimum $50,000 capital** per trade for profitability. Typical net profit: 0.3-0.7% per trade.

---

## 1. Role

**Cross-chain arbitrage analyst** -- identifies and evaluates price differentials between the same asset on different blockchains, incorporating bridge logistics and CEX hedge mechanics.

This skill is responsible for:
- Scanning multiple chain pairs for cross-chain price differentials
- Computing all-in cost including gas (both chains), bridge fee, slippage (both DEXes), and hedge cost
- Comparing bridge protocols by speed, fee, and reliability
- Designing transit hedge positions using OKX perpetual shorts
- Producing ranked, actionable recommendations with full cost breakdowns

This skill does **NOT**:
- Execute any trades (analysis only, never sends orders or signs transactions)
- Initiate bridge transfers or onchain transactions
- Open or close positions on any CEX
- Handle single-chain CEX-DEX arbitrage (delegate to `cex-dex-arbitrage`)
- Handle funding rate carry (delegate to `funding-rate-arbitrage`)
- Handle basis/delivery futures (delegate to `basis-trading`)
- Handle DeFi yield farming (delegate to `yield-optimizer`)

**Key Principle:** Every recommendation passes through GoPlus security checks on BOTH source and destination chains, bridge safety validation, AND a full cost analysis before being output. If any gate fails, the opportunity is suppressed with a clear explanation.

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Technical labels may remain in English regardless of language:
- `bps`, `spread`, `bridge`, `hedge`, `transit`, `PnL`, `slippage`, `gas`, `gwei`, `taker`, `maker`, `TVL`
- Bridge names always in English: Across, Stargate, Synapse
- Chain names always in English: Ethereum, Arbitrum, Base, Optimism, Polygon, Solana, BSC
- Timestamps always displayed in UTC

Examples:
- User writes "scan for cross-chain arb" --> respond in English
- User writes "跨鏈套利機會" --> respond in Traditional Chinese
- User writes "有冇跨鏈搬磚機會" --> respond in Cantonese-style Traditional Chinese
- User writes "跨链套利分析" --> respond in Simplified Chinese

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations -- no trades, no transfers, no bridge calls |
| Recommendation header | Always show `[RECOMMENDATION ONLY -- 不會自動執行]` |
| Live switch | Requires explicit user confirmation (see Account Safety Protocol in Section 12) |
| Capital requirement | Always display `[MIN CAPITAL: $50,000]` warning when position size < $50K |

Even in `[LIVE]` mode, this skill only reads market data and produces recommendations. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

This skill requires **both** CEX and DEX venues across **multiple chains**. All three infrastructure layers (OKX MCP, OnchainOS, GoPlus) are mandatory.

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK -- output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN -- display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | SWAP instruments accessible | `market_get_instruments(instType: "SWAP")` | Returns non-empty array of perpetual swap instruments | WARN -- hedge unavailable. Proceed but label ALL outputs `[UNHEDGED -- NO PERP AVAILABLE]`. |
| 4 | OnchainOS CLI installed | `which onchainos` (Bash) | Exit code 0, returns a valid path | BLOCK -- DEX venue required for cross-chain arb. Tell user: `npx skills add okx/onchainos-skills` |
| 5 | OnchainOS CLI functional (source chain) | `onchainos dex-market price --chain {source_chain} --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | Returns valid JSON with `priceUsd` field | BLOCK -- Source chain DEX data required. Suggest checking network connectivity. |
| 6 | OnchainOS CLI functional (dest chain) | `onchainos dex-market price --chain {dest_chain} --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | Returns valid JSON with `priceUsd` field | BLOCK -- Destination chain DEX data required. Suggest checking network connectivity. |
| 7 | GoPlus MCP available | `check_token_security` with a known-good token (e.g., WETH on Ethereum) | Returns valid security object | WARN -- security checks unavailable. Proceed but label ALL outputs `[SECURITY UNCHECKED]`. |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX hedge data)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> WARN: hedge unavailable, continue with unhedged labels
    Check 3 PASS -> Check 4
      Check 4 FAIL -> BLOCK (DEX venue mandatory for cross-chain arb)
      Check 4 PASS -> Check 5
        Check 5 FAIL -> BLOCK (source chain inaccessible)
        Check 5 PASS -> Check 6
          Check 6 FAIL -> BLOCK (dest chain inaccessible)
          Check 6 PASS -> Check 7
            Check 7 FAIL -> WARN: security unchecked, continue with labels
            Check 7 PASS -> ALL SYSTEMS GO
```

Unlike `cex-dex-arbitrage`, this skill requires DEX data from **two** chains simultaneously and SWAP instruments for the hedge leg.

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "跨鏈套利機會" / "cross-chain arb opportunities" | Yes -- `scan` | -- |
| "ETH 喺 Arbitrum 同 Base 價差幾多" / "ETH spread across L2s" | Yes -- `evaluate` | -- |
| "邊條橋最快最平" / "which bridge is cheapest" | Yes -- `bridge-compare` | -- |
| "橋接期間點對沖" / "how to hedge during bridge" | Yes -- `transit-hedge` | -- |
| "跨鏈搬磚" / "cross-chain搬磚" | Yes -- `scan` (default command for cross-chain arb intent) | -- |
| "CEX 同 DEX 價差" / "CEX-DEX spread" (single chain) | No | `cex-dex-arbitrage` |
| "BTC 而家幾錢" / "what's the BTC price" | No | `price-feed-aggregator.snapshot` |
| "資金費率套利" / "funding rate arb" | No | `funding-rate-arbitrage` |
| "基差交易" / "basis trade" | No | `basis-trading` |
| "穩定幣脫錨" / "stablecoin depeg" | No | `stablecoin-depeg-arbitrage` |
| "邊度收益最好" / "best yield" | No | `yield-optimizer` |
| "聰明錢買咗乜" / "smart money" | No | `smart-money-tracker` |
| "幫我買" / "execute trade" | No | Refuse -- no skill executes trades |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Multi-chain price differential scanner | Read | Scan multiple assets across multiple chain pairs for profitable cross-chain spreads |
| `evaluate` | Single-opportunity deep analysis | Read | Full cost breakdown, security check, bridge selection, hedge design for one specific opportunity |
| `bridge-compare` | Bridge protocol comparison | Read | Compare bridge options for a specific transfer (speed, cost, TVL, reliability) |
| `transit-hedge` | Hedge design calculator | Read | Design the optimal CEX perpetual hedge position for a specific bridge transfer |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Scan multiple assets across multiple chain pairs for profitable cross-chain price differentials. Returns a ranked list of opportunities filtered by minimum spread, capital, and bridge availability.

```bash
cross-chain-arbitrage scan --assets ETH,WBTC --source-chains ethereum,arbitrum --dest-chains base,optimism --min-spread-bps 100 --size-usd 50000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--assets` | string[] | No | `["ETH"]` | Any valid cross-chain asset | Uppercase, comma-separated, max 10 items. Must exist on both source and dest chains. |
| `--source-chains` | string[] | No | `["ethereum"]` | `ethereum`, `arbitrum`, `base`, `optimism`, `polygon`, `bsc`, `solana` | Must be supported chains (see Chain Reference). |
| `--dest-chains` | string[] | No | `["arbitrum","base"]` | Same as source-chains | Must be supported. Cannot overlap with source-chains. |
| `--min-spread-bps` | number | No | `100` | -- | Min: 50, Max: 2000. Only show opportunities above this raw spread. |
| `--size-usd` | number | No | `50000` | -- | Min: 10,000, Max: 500,000. Trade size for cost estimation. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`, `VIP1`, `VIP2`, `VIP3`, `VIP4`, `VIP5` | OKX fee tier for hedge cost calculation. |
| `--bridge-preference` | string | No | `"auto"` | `auto`, `across`, `stargate`, `synapse` | Preferred bridge protocol. `auto` selects optimal. |
| `--max-bridge-time-min` | number | No | `10` | -- | Min: 1, Max: 30. Maximum acceptable bridge transit time. |
| `--skip-security` | boolean | No | `false` | `true`, `false` | Skip GoPlus checks. Output labeled `[SECURITY UNCHECKED]`. |

#### Return Schema

```yaml
CrossChainScanResult:
  timestamp: integer              # Unix ms when scan completed
  mode: string                    # "demo" or "live"
  scan_params:
    assets: string[]
    source_chains: string[]
    dest_chains: string[]
    min_spread_bps: number
    size_usd: number
    vip_tier: string
    bridge_preference: string
  total_pairs_scanned: integer    # Total asset x chain-pair combinations
  opportunities_found: integer    # Pairs passing all filters
  opportunities:
    - rank: integer               # 1 = best net profit
      asset: string               # e.g. "ETH"
      source_chain: string        # e.g. "ethereum"
      dest_chain: string          # e.g. "arbitrum"
      source_dex_price: string    # Price on source chain DEX
      dest_dex_price: string      # Price on dest chain DEX
      cex_price: string           # OKX price (for hedge reference)
      gross_spread_bps: number    # Raw spread before costs
      direction: string           # "Buy {source} / Sell {dest}" or vice versa
      recommended_bridge: string  # e.g. "Across"
      bridge_time_min: string     # e.g. "1-4"
      total_costs_usd: number     # Sum of all cost layers
      net_profit_usd: number      # After all costs at size_usd
      net_profit_pct: number      # net_profit / size_usd * 100
      profit_to_cost_ratio: number
      hedge_available: boolean    # Whether OKX perp exists for hedge
      security_status: string     # "SAFE", "WARN", "BLOCK", "UNCHECKED"
      security_warnings: string[]
      risk_score: integer         # 1-10
      confidence: string          # "high", "medium", "low"
  blocked_assets:
    - asset: string
      source_chain: string
      dest_chain: string
      reason: string              # e.g. "BRIDGE_TVL_LOW: Across TVL $5M < $10M minimum"
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `total_pairs_scanned` | integer | Number of asset x (source_chain, dest_chain) combinations evaluated |
| `opportunities_found` | integer | Opportunities with net_profit > 0 and spread > min_spread_bps |
| `gross_spread_bps` | number | `abs(source_price - dest_price) / min(source_price, dest_price) * 10000` |
| `direction` | string | Which chain is cheaper: buy there, bridge, sell on the other |
| `net_profit_usd` | number | After ALL costs: gas x2, bridge, slippage x2, hedge open/close |
| `hedge_available` | boolean | Whether a matching OKX perpetual swap exists for transit hedging |

---

### 7.2 Command: `evaluate`

Deep analysis of a single cross-chain arbitrage opportunity. Performs GoPlus security audit on both chains, computes bridge options, designs hedge position, and produces full profitability calculation.

```bash
cross-chain-arbitrage evaluate --asset ETH --source-chain ethereum --dest-chain arbitrum --size-usd 50000 --bridge across
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid cross-chain asset | Uppercase, single asset. |
| `--source-chain` | string | Yes | -- | `ethereum`, `arbitrum`, `base`, `optimism`, `polygon`, `bsc`, `solana` | The chain where the asset is cheaper (buy side). |
| `--dest-chain` | string | Yes | -- | Same as source-chain | The chain where the asset is expensive (sell side). Must differ from source. |
| `--size-usd` | number | No | `50000` | -- | Min: 10,000, Max: 500,000. Trade size. |
| `--bridge` | string | No | `"auto"` | `auto`, `across`, `stargate`, `synapse` | Bridge to use. `auto` selects optimal. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0` through `VIP5` | OKX fee tier for hedge cost. |
| `--contract-address-source` | string | No | auto-resolve | -- | Explicit token address on source chain. |
| `--contract-address-dest` | string | No | auto-resolve | -- | Explicit token address on dest chain. |
| `--hedge-leverage` | number | No | `1` | -- | Min: 1, Max: 3. Hedge position leverage. Higher = less capital tied up, more risk. |

#### Return Schema

```yaml
CrossChainEvaluateResult:
  timestamp: integer
  mode: string
  asset: string
  source_chain: string
  dest_chain: string
  size_usd: number
  contract_address_source: string
  contract_address_dest: string

  prices:
    source_dex:
      price: string
      source: string              # "dex-market price --chain {source}"
      data_age_ms: integer
    dest_dex:
      price: string
      source: string              # "dex-market price --chain {dest}"
      data_age_ms: integer
    cex:
      last: string
      bid: string
      ask: string
      source: string              # "market_get_ticker"
      data_age_ms: integer

  spread:
    gross_spread_bps: number
    gross_spread_usd: number
    direction: string
    buy_chain: string
    sell_chain: string

  security:
    source_chain:
      status: string              # "SAFE", "WARN", "BLOCK"
      warnings: string[]
    dest_chain:
      status: string
      warnings: string[]
    overall: string

  bridge:
    selected: string              # Bridge name
    estimated_time_min: string    # e.g. "1-4"
    fee_pct: number               # e.g. 0.001
    fee_usd: number
    tvl_usd: number               # Bridge pool TVL
    recent_exploits: boolean
    alternatives:                  # Other bridge options evaluated
      - name: string
        time_min: string
        fee_pct: number
        fee_usd: number

  hedge:
    available: boolean
    instId: string                # e.g. "ETH-USDT-SWAP"
    position_side: string         # "short" (always, to hedge long during bridge)
    position_size_tokens: number
    position_size_usd: number
    leverage: number
    open_fee_usd: number          # Taker fee to open
    close_fee_usd: number         # Taker fee to close
    estimated_funding_cost: number  # Funding for ~5 min (negligible)
    total_hedge_cost: number

  cost_breakdown:
    source_chain:
      gas_cost_usd: number
      dex_slippage_usd: number
      dex_slippage_bps: number
      subtotal: number
    dest_chain:
      gas_cost_usd: number
      dex_slippage_usd: number
      dex_slippage_bps: number
      subtotal: number
    bridge:
      fee_usd: number
    hedge:
      open_close_fees: number
      funding_cost: number
      subtotal: number
    total_costs_usd: number
    total_costs_bps: number       # As proportion of size_usd

  profitability:
    gross_spread_usd: number
    total_costs_usd: number
    net_profit_usd: number
    net_profit_pct: number
    profit_to_cost_ratio: number
    min_spread_for_breakeven_bps: number
    is_profitable: boolean
    confidence: string

  risk:
    overall_score: integer        # 1-10
    risk_gauge: string            # "▓▓▓░░░░░░░"
    execution_risk: integer       # Bridge failure, timing
    market_risk: integer          # Price movement during transit
    contract_risk: integer        # Token security
    bridge_risk: integer          # Bridge protocol risk
    liquidity_risk: integer       # DEX depth on both chains
    hedge_risk: integer           # Hedge availability and cost

  recommended_action: string      # "PROCEED", "CAUTION", "DO NOT TRADE"
  next_steps: string[]
```

---

### 7.3 Command: `bridge-compare`

Compare bridge protocols for a specific cross-chain transfer. Evaluates speed, cost, TVL, and recent security history.

```bash
cross-chain-arbitrage bridge-compare --asset ETH --source-chain ethereum --dest-chain arbitrum --size-usd 50000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any bridgeable asset | Uppercase, single asset. |
| `--source-chain` | string | Yes | -- | See chain reference | Origin chain. |
| `--dest-chain` | string | Yes | -- | See chain reference | Destination chain. Must differ from source. |
| `--size-usd` | number | No | `50000` | -- | Min: 100, Max: 500,000. Transfer size for fee estimation. |

#### Return Schema

```yaml
BridgeCompareResult:
  timestamp: integer
  asset: string
  source_chain: string
  dest_chain: string
  size_usd: number

  bridges:
    - name: string                # e.g. "Across"
      available: boolean
      estimated_time_min: string  # e.g. "1-4"
      fee_pct: number
      fee_usd: number
      tvl_usd: number             # Pool TVL for this route
      min_transfer: number        # Minimum transfer in USD
      max_transfer: number        # Maximum single transfer
      recent_exploits_90d: boolean
      reliability_score: string   # "high", "medium", "low"
      recommended: boolean        # Whether this is the top pick

  recommendation:
    best_bridge: string
    reason: string
    warnings: string[]

  never_use:
    - name: string
      reason: string              # e.g. "Official L2 bridge: 7-day withdrawal delay"
```

---

### 7.4 Command: `transit-hedge`

Design the optimal CEX perpetual hedge position for a specific bridge transfer. Calculates exact position size, expected duration, cost, and risk.

```bash
cross-chain-arbitrage transit-hedge --asset ETH --size-usd 50000 --bridge-time-min 3 --vip-tier VIP1
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any asset with OKX perp | Uppercase. Must have `-USDT-SWAP` instrument. |
| `--size-usd` | number | Yes | -- | -- | Min: 10,000, Max: 500,000. Bridge transfer size. |
| `--bridge-time-min` | number | No | `5` | -- | Min: 1, Max: 30. Expected bridge transit time. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0` through `VIP5` | OKX fee tier. |
| `--leverage` | number | No | `1` | -- | Min: 1, Max: 3. Hedge leverage. |

#### Return Schema

```yaml
TransitHedgeResult:
  timestamp: integer
  asset: string
  instId: string                  # e.g. "ETH-USDT-SWAP"

  hedge_design:
    side: string                  # "short" (always short to hedge long transit)
    size_tokens: number
    size_usd: number
    leverage: number
    margin_required: number       # size_usd / leverage
    estimated_duration_min: number

  costs:
    open_fee: number              # Taker fee to open short
    close_fee: number             # Taker fee to close short
    funding_periods: number       # Number of 8h funding periods (usually 0 for 5 min)
    estimated_funding_cost: number
    slippage_open: number
    slippage_close: number
    total_hedge_cost: number

  current_market:
    funding_rate_8h: string
    funding_direction: string     # "longs pay shorts" or "shorts pay longs"
    next_funding_time: integer
    perp_mark_price: string
    orderbook_depth_at_size: number

  risk_assessment:
    hedge_effectiveness: string   # "full", "partial", "none"
    unhedged_exposure_pct: number # Basis risk between perp and spot
    max_loss_if_hedge_fails: number
    warnings: string[]

  recommendation: string          # "HEDGE RECOMMENDED", "HEDGE OPTIONAL", "CANNOT HEDGE"
```

---

## 8. Execution Flow

### Step 1: Intent Recognition

Parse user message to extract command, parameters, and context.

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `bridge-compare` / `transit-hedge` based on keywords | Default: `scan` |
| Assets | Extract token symbols (ETH, WBTC, USDC, SOL...) | Default: `["ETH"]` |
| Source chains | Look for chain names after "from", "源", "喺" | Default: `["ethereum"]` |
| Dest chains | Look for chain names after "to", "去", "目標" | Default: `["arbitrum","base"]` |
| Size | Look for USD amounts: "$50K", "50000 USDT", "五萬" | Default: `50000` |
| Min spread | Look for bps values: "100 bps", "1%", "100 基點" | Default: `100` (scan) |
| Bridge | Look for bridge names: "Across", "Stargate", "Synapse" | Default: `"auto"` |
| VIP tier | Look for "VIP1", "VIP2" etc. | Default: `"VIP0"` |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "搵", "找", "跨鏈搬磚", "cross-chain scan", "機會", "opportunities" | `scan` |
| "評估", "evaluate", "分析", "detailed", "值唔值得", "profitable", "跨鏈分析" | `evaluate` |
| "橋", "bridge", "compare", "比較", "邊條橋", "which bridge", "最快", "最平" | `bridge-compare` |
| "對沖", "hedge", "transit", "橋接對沖", "protect", "保護", "short during bridge" | `transit-hedge` |

**Ambiguous intent resolution:**

| Input Pattern | Resolved Command | Reasoning |
|---------------|-----------------|-----------|
| "跨鏈搬磚" (generic) | `scan` | Broad cross-chain arb intent -> scan with defaults |
| "ETH Arbitrum 同 Base 有冇價差" | `evaluate --asset ETH --source-chain arbitrum --dest-chain base` | Specific asset + chains -> evaluate |
| "Across 定 Stargate 好啲" | `bridge-compare` | Bridge comparison intent |
| "橋接期間點對沖" | `transit-hedge` | Hedge design intent |

---

### Step 2: Pre-Execution Safety Checks

For each candidate token and chain pair, run these checks **before any price analysis**:

#### 2a. Token Address Resolution (Both Chains)

```
For each (asset, source_chain, dest_chain):

  FOR EACH chain IN [source_chain, dest_chain]:

    IF asset is native token on chain (ETH on ethereum/arbitrum/base/optimism, SOL on solana, BNB on bsc, MATIC on polygon):
      -> Use native address: 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee (EVM) or native (Solana)
      -> SKIP GoPlus security check (native tokens are safe)

    ELSE:
      -> onchainos dex-token search {ASSET} --chains {chain}
      -> Pick result matching target chain with highest liquidity
      -> If no results -> BLOCK: TOKEN_NOT_FOUND for this asset/chain
      -> Store: token_address_cache[{ASSET}:{CHAIN}] = address
```

#### 2b. GoPlus Security Check (Both Chains, Non-Native Tokens)

```
For each non-native token on EACH chain:

  1. GoPlus check_token_security(chain_id, contract_address)

  BLOCK conditions (any single trigger -> remove from scan):
    - is_honeypot === "1"           -> BLOCK("Honeypot detected on {chain}")
    - buy_tax > 0.05 (5%)          -> BLOCK("Buy tax {tax}% exceeds 5% limit on {chain}")
    - sell_tax > 0.10 (10%)        -> BLOCK("Sell tax {tax}% exceeds 10% limit on {chain}")
    - is_open_source === "0"       -> BLOCK("Contract not verified on {chain}")
    - can_take_back_ownership === "1" -> BLOCK("Ownership reclaimable on {chain}")
    - owner_change_balance === "1"  -> BLOCK("Owner can modify balances on {chain}")
    - slippage_modifiable === "1"  -> BLOCK("Tax modifiable by owner on {chain}")
    - cannot_sell_all === "1"      -> BLOCK("Cannot sell entire balance on {chain}")

  WARN conditions (flag but continue):
    - is_proxy === "1"             -> WARN("Upgradeable proxy on {chain}")
    - buy_tax > 0.01 (1%)         -> WARN("Buy tax > 1% on {chain}")
    - sell_tax > 0.02 (2%)        -> WARN("Sell tax > 2% on {chain}")
    - is_mintable === "1"          -> WARN("Token is mintable on {chain}")

  IMPORTANT: Token must pass security on BOTH chains. A BLOCK on either chain blocks the opportunity.
```

#### 2c. Bridge Safety Validation

```
For each bridge candidate:

  Check 1: Bridge TVL
    IF bridge_tvl_usd < 10,000,000 ($10M):
      -> BLOCK: BRIDGE_TVL_LOW
      -> Message: "Bridge TVL ${tvl} is below $10M minimum -- liquidity risk too high"

  Check 2: Recent Exploits
    IF bridge has had an exploit or security incident in last 90 days:
      -> BLOCK: BRIDGE_EXPLOIT_RECENT
      -> Message: "Bridge had a security incident on {date}. Avoid for 90 days."

  Check 3: Bridge Speed
    IF estimated_time > max_bridge_time_min (default 10 min):
      -> BLOCK for arbitrage: BRIDGE_TOO_SLOW
      -> Message: "Bridge time {time} min exceeds {max} min limit for arbitrage"

  NEVER USE these bridges for arbitrage:
    - Official L2 bridges (7-day withdrawal for optimistic rollups)
    - Any bridge with > 1% fee
    - Any bridge with > 30 min estimated time
```

#### 2d. Liquidity Validation (Both Chains)

```
For each token on EACH chain:

  onchainos dex-token price-info {contract_address} --chain {chain}
  -> Extract: liquidityUsd

  IF liquidityUsd < 500,000 ($500K):
    -> BLOCK: INSUFFICIENT_LIQUIDITY
    -> Message: "DEX liquidity on {chain} is ${actual}, minimum $500K for cross-chain arb at $50K+"

  IF liquidityUsd < 2,000,000 ($2M):
    -> WARN: "Low liquidity on {chain} -- slippage may be elevated at $50K+ size"
```

#### 2e. Hedge Availability Check

```
For the asset being traded:

  market_get_instruments(instType: "SWAP")
  -> Search for {ASSET}-USDT-SWAP

  IF not found:
    -> WARN: NO_HEDGE_AVAILABLE
    -> Message: "No OKX perpetual swap for {ASSET}. Bridge transit is UNHEDGED -- price may move 1-5% during 1-5 min transit."
    -> Risk score += 3

  IF found:
    -> Verify orderbook depth: market_get_orderbook(instId: "{ASSET}-USDT-SWAP")
    -> IF depth at size_usd < 80% of position:
      -> WARN: "Hedge orderbook thin -- may experience significant slippage"
```

---

### Step 3: Data Collection & Computation

#### 3a. Cross-Chain Price Snapshot

```
For each (asset, source_chain, dest_chain) that passed Step 2:

  1. Source chain DEX price:
     onchainos dex-market price --chain {source_chain} --token {source_address}
     -> Extract: priceUsd (source_dex_price)

  2. Destination chain DEX price:
     onchainos dex-market price --chain {dest_chain} --token {dest_address}
     -> Extract: priceUsd (dest_dex_price)

  3. CEX reference price (for hedge):
     market_get_ticker({ instId: "{ASSET}-USDT" })
     -> Extract: last, bidPx, askPx, ts (cex_price)
```

#### 3b. Cross-Chain Spread Calculation

```
spread_bps = abs(source_dex_price - dest_dex_price) / min(source_dex_price, dest_dex_price) * 10000

IF spread_bps < min_spread_bps:
  -> Skip this pair (below threshold)

IF spread_bps > 5000 (50%):
  -> BLOCK: PRICE_ANOMALY
  -> Likely wrong token address or dead pool on one chain

Determine direction:
  IF source_dex_price < dest_dex_price:
    direction = "Buy {source_chain} / Bridge / Sell {dest_chain}"
    buy_chain = source_chain
    sell_chain = dest_chain
  ELSE:
    direction = "Buy {dest_chain} / Bridge / Sell {source_chain}"
    buy_chain = dest_chain
    sell_chain = source_chain
```

#### 3c. All-In Cost Calculation

```
total_cost =
  gas_source_chain          (swap on source DEX)
  + bridge_fee              (bridge protocol fee)
  + gas_dest_chain          (swap on destination DEX)
  + slippage_source         (from DEX quote on source chain)
  + slippage_dest           (from DEX quote on dest chain)
  + hedge_open_fee          (OKX taker fee to open short)
  + hedge_close_fee         (OKX taker fee to close short)
  + hedge_funding           (funding cost for ~5 min, usually negligible)

Each component:

  gas_source = onchainos dex-swap quote on source chain -> estimatedGas * gasPrice * nativeTokenPrice
  gas_dest = onchainos dex-swap quote on dest chain -> estimatedGas * gasPrice * nativeTokenPrice

  bridge_fee = size_usd * bridge_fee_rate
    Across (stables): 0.001% = $0.50 per $50K
    Across (ETH): 0.01% = $5.00 per $50K
    Stargate V2: 0.06% = $30.00 per $50K
    Synapse: 0.04% = $20.00 per $50K

  slippage_source = size_usd * estimated_slippage_bps / 10000
  slippage_dest = size_usd * estimated_slippage_bps / 10000

  hedge_open_fee = size_usd * swap_taker_rate (from Fee Schedule)
  hedge_close_fee = size_usd * swap_taker_rate
  hedge_funding = size_usd * abs(funding_rate_8h) * (bridge_time_min / 480)
    (480 min = 8 hours; for 5 min transit this is ~0.001% -- negligible)
```

#### 3d. Profitability Determination

```
gross_spread_usd = size_usd * spread_bps / 10000
net_profit_usd = gross_spread_usd - total_cost
net_profit_pct = net_profit_usd / size_usd * 100
profit_to_cost_ratio = net_profit_usd / total_cost
min_spread_for_breakeven_bps = total_cost / size_usd * 10000

is_profitable = net_profit_usd > 0 AND profit_to_cost_ratio > 1.0
```

#### 3e. Risk Scoring

```
risk_score components (each 1-10, weighted average):

  execution_risk (weight: 0.25):
    Base: 4 (cross-chain inherently riskier than single-chain)
    +1 if bridge_time > 5 min
    +2 if bridge_time > 10 min
    +1 if hedge unavailable

  market_risk (weight: 0.20):
    Base: 3
    +1 per 2% of 24h price range
    +2 if unhedged bridge transit

  contract_risk (weight: 0.15):
    1 if native token on both chains
    3 if verified ERC-20 on both chains
    6 if any WARN from GoPlus on either chain

  bridge_risk (weight: 0.20):
    2 if Across (battle-tested)
    3 if Stargate V2
    4 if Synapse
    +2 if bridge_tvl < $50M
    +3 if bridge_tvl < $20M

  liquidity_risk (weight: 0.10):
    1 if liquidity > $50M on both chains
    3 if liquidity > $10M on both chains
    6 if liquidity > $2M on both chains
    8 if liquidity < $2M on either chain

  hedge_risk (weight: 0.10):
    1 if fully hedged with deep orderbook
    4 if hedged but thin orderbook
    8 if no hedge available

  overall_risk = sum(component * weight), rounded to nearest integer
```

---

### Step 4: Output & Recommend

Format using output templates (see Section 13). The output structure varies by command:

#### For `scan`:
1. Global header (skill name, mode, timestamp, data sources)
2. Scan summary (parameters, total scanned, opportunities found)
3. Opportunity table (ranked, with key metrics per row)
4. Blocked pairs section (if any failed security/bridge/liquidity)
5. Risk gauge for top opportunity
6. Next steps suggestions
7. Disclaimer

#### For `evaluate`:
1. Global header
2. Price comparison (source DEX vs dest DEX vs CEX reference)
3. Spread analysis (gross, direction, chains)
4. Security check results (GoPlus on both chains)
5. Bridge selection with alternatives
6. Hedge design summary
7. Cost breakdown table (all 6 cost layers)
8. Net profit summary
9. Risk gauge (6-dimension breakdown)
10. Recommended action
11. Execution sequence timeline
12. Next steps
13. Disclaimer

#### For `bridge-compare`:
1. Global header
2. Bridge comparison table
3. Recommended bridge with reasoning
4. Never-use list
5. Disclaimer

#### For `transit-hedge`:
1. Global header
2. Hedge position design
3. Cost breakdown
4. Market conditions
5. Risk assessment
6. Disclaimer

**Suggested follow-up actions (vary by result):**

| Result | Suggested Actions |
|--------|-------------------|
| Scan: 0 opportunities | "暫無跨鏈套利機會。鏈間價差通常在 CEX 維護或網絡擁堵時出現。" |
| Scan: opportunities found | "詳細評估最佳機會 -> `cross-chain-arbitrage evaluate --asset {TOP} --source-chain {SRC} --dest-chain {DST}`" |
| Evaluate: profitable | "如滿意，手動按以下順序執行 5 步交易序列 (見執行時間線)"; "比較不同橋 -> `cross-chain-arbitrage bridge-compare`" |
| Evaluate: not profitable | "價差不足以覆蓋成本。等待更大價差或嘗試不同鏈對。" |
| Bridge-compare | "使用推薦橋評估完整機會 -> `cross-chain-arbitrage evaluate --bridge {BEST}`" |

---

## 9. Key Formulas (Inlined)

### 9.1 Cross-Chain Spread (basis points)

```
spread_bps = abs(price_chain_a - price_chain_b) / min(price_chain_a, price_chain_b) * 10000
```

| Variable | Definition |
|----------|-----------|
| `price_chain_a` | DEX price on chain A (from OnchainOS `dex-market price`) |
| `price_chain_b` | DEX price on chain B (from OnchainOS `dex-market price`) |
| `spread_bps` | Unsigned spread in basis points (1 bp = 0.01%) |

**Worked Example**

```
price_ethereum = $3,412.50  (ETH price on Ethereum Uniswap)
price_arbitrum = $3,446.22  (ETH price on Arbitrum Uniswap)

spread_bps = abs(3412.50 - 3446.22) / min(3412.50, 3446.22) * 10000
           = 33.72 / 3412.50 * 10000
           = 98.8 bps (~0.99%)
```

### 9.2 All-In Cost Calculation

```
total_cost = gas_source + bridge_fee + gas_dest + slippage_source + slippage_dest + hedge_cost

Where:
  gas_source = gas_price_gwei * gas_limit * native_price / 1e9         (EVM chains)
  bridge_fee = size_usd * bridge_fee_rate
  gas_dest = gas_price_gwei * gas_limit * native_price / 1e9
  slippage_source = size_usd * estimated_slippage_bps / 10000
  slippage_dest = size_usd * estimated_slippage_bps / 10000
  hedge_cost = (size_usd * swap_taker_rate * 2) + funding_cost_during_transit
```

**Worked Example ($50,000 ETH, Ethereum -> Arbitrum via Across)**

```
Gas source (Ethereum swap, 30 gwei, 200K gas, ETH=$3,412):
  = 30 * 200,000 * 3,412 / 1,000,000,000
  = $20.47

Bridge fee (Across, ETH, 0.01%):
  = 50,000 * 0.0001
  = $5.00

Gas dest (Arbitrum swap, 0.25 gwei, 1,500,000 gas, ETH=$3,412):
  = 0.25 * 1,500,000 * 3,412 / 1,000,000,000
  = $1.28

Slippage source (Ethereum, estimated 10 bps at $50K):
  = 50,000 * 10 / 10,000
  = $50.00

Slippage dest (Arbitrum, estimated 30 bps at $50K):
  = 50,000 * 30 / 10,000
  = $150.00

Hedge cost (open + close short on OKX, VIP0 SWAP taker 0.05%):
  open_fee  = 50,000 * 0.0005 = $25.00
  close_fee = 50,000 * 0.0005 = $25.00
  funding (5 min of 0.01% per 8h) = 50,000 * 0.0001 * (5/480) = $0.05
  hedge_total = $50.05

TOTAL COST = $20.47 + $5.00 + $1.28 + $50.00 + $150.00 + $50.05
           = $276.80
           = 55.4 bps of $50,000

Breakeven spread = 55.4 bps
Profitable if spread > 55.4 bps + profit_margin (20 bps) = ~75.4 bps
```

**Worked Example ($50,000 ETH, Arbitrum -> Base via Across)**

```
Gas source (Arbitrum swap): $0.35
Bridge fee (Across, ETH, 0.01%): $5.00
Gas dest (Base swap): $0.03
Slippage source (Arbitrum, 20 bps): $100.00
Slippage dest (Base, 30 bps): $150.00
Hedge cost (VIP0): $50.05

TOTAL COST = $0.35 + $5.00 + $0.03 + $100.00 + $150.00 + $50.05
           = $305.43
           = 61.1 bps of $50,000
```

### 9.3 Net Profit

```
net_profit = gross_spread_usd - total_cost
gross_spread_usd = size_usd * spread_bps / 10000
net_profit_pct = net_profit / size_usd * 100
```

**Worked Example (continuing from 9.2)**

```
spread_bps = 98.8
size_usd = $50,000

gross_spread_usd = 50,000 * 98.8 / 10,000 = $494.00
net_profit = $494.00 - $276.80 = $217.20
net_profit_pct = $217.20 / $50,000 * 100 = 0.43%
profit_to_cost_ratio = $217.20 / $276.80 = 0.78x
```

### 9.4 Hedge Position Sizing

```
hedge_size_tokens = bridge_amount_usd / cex_price
hedge_size_contracts = hedge_size_tokens / contract_size

For OKX USDT-margined perps:
  contract_size is typically 1 token for most assets
  hedge_size = bridge_amount_usd (in USDT terms)
```

**Worked Example**

```
bridge_amount_usd = $50,000
cex_price (ETH) = $3,412.50
contract_size = 1 ETH per contract (USDT-margined)

hedge_size_tokens = 50,000 / 3,412.50 = 14.65 ETH
hedge_size_contracts = 14.65 contracts (round to 14.6 or 14.7)
margin_required (1x) = $50,000
margin_required (3x) = $16,667
```

### 9.5 Hedge Duration and Funding Cost

```
funding_cost = size_usd * abs(funding_rate_8h) * (bridge_time_min / 480)

Where:
  funding_rate_8h = current 8-hour funding rate (e.g., 0.0001 = 0.01%)
  bridge_time_min = estimated bridge transit time in minutes
  480 = minutes in 8 hours (one funding period)
```

**Worked Example**

```
size_usd = $50,000
funding_rate_8h = 0.0001 (0.01%)
bridge_time_min = 3 minutes

funding_cost = 50,000 * 0.0001 * (3 / 480)
             = 50,000 * 0.0001 * 0.00625
             = $0.03

Conclusion: Funding cost during bridge transit is negligible (<$1 in virtually all cases).
```

### 9.6 Minimum Capital for Profitability

```
min_capital = fixed_costs / (target_spread_pct - variable_cost_pct)

Where:
  fixed_costs = gas_source + gas_dest + bridge_base_fee
  variable_cost_pct = slippage_pct + hedge_fee_pct
  target_spread_pct = expected spread as decimal
```

**Worked Example (Ethereum -> Arbitrum)**

```
fixed_costs = $20.47 (gas source) + $1.28 (gas dest) + $5.00 (bridge base) = $26.75
variable_cost_pct = 0.004 (40 bps slippage) + 0.001 (10 bps hedge) = 0.005
target_spread_pct = 0.01 (100 bps = 1%)

min_capital = $26.75 / (0.01 - 0.005)
            = $26.75 / 0.005
            = $5,350

For 0.2% profit margin:
min_capital = $26.75 / (0.01 - 0.005 - 0.002)
            = $26.75 / 0.003
            = $8,917

Note: This is theoretical minimum. Practical minimum is $50,000 because:
  1. Slippage scales non-linearly (worse at smaller sizes on thin pools)
  2. Need enough profit to justify execution complexity
  3. Bridge minimum amounts may apply
```

### 9.7 Signal Decay for Cross-Chain Arbitrage

```
signal_strength = initial_strength * exp(-decay_rate * time_elapsed_minutes)

Cross-chain arb decay rate: 0.10 (slower than single-chain CEX-DEX arb at 0.15)
Half-life: ~6.9 minutes
```

**Worked Example**

```
initial_strength = 0.95  (strong cross-chain signal)
decay_rate = 0.10
time_elapsed = 8 minutes

signal_strength = 0.95 * exp(-0.10 * 8)
               = 0.95 * 0.4493
               = 0.427

Signal has decayed from 0.95 to 0.43 in 8 minutes.
Cross-chain signals decay slower than single-chain because:
  - Fewer arbitrageurs have cross-chain infrastructure
  - Bridge transit itself creates a natural barrier
```

---

## 10. Safety Checks (Per Operation)

### 10.1 Pre-Trade Safety Checklist

Every command runs these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | -- | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | -- | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 10s stale (cross-chain needs more time) | > 5s stale | `DATA_STALE` |
| 4 | Token honeypot (source chain) | GoPlus `check_token_security` | `is_honeypot === "1"` | -- | `SECURITY_BLOCKED_SOURCE` |
| 5 | Token honeypot (dest chain) | GoPlus `check_token_security` | `is_honeypot === "1"` | -- | `SECURITY_BLOCKED_DEST` |
| 6 | Token tax rate (both chains) | GoPlus `check_token_security` | buy > 5% OR sell > 10% | buy > 1% | `SECURITY_BLOCKED` |
| 7 | Contract verified (both chains) | GoPlus `check_token_security` | `is_open_source === "0"` | -- | `SECURITY_BLOCKED` |
| 8 | Source chain DEX liquidity | OnchainOS `dex-token price-info` | `liquidityUsd < $500,000` | `liquidityUsd < $2,000,000` | `INSUFFICIENT_LIQUIDITY_SOURCE` |
| 9 | Dest chain DEX liquidity | OnchainOS `dex-token price-info` | `liquidityUsd < $500,000` | `liquidityUsd < $2,000,000` | `INSUFFICIENT_LIQUIDITY_DEST` |
| 10 | Bridge TVL | Bridge API / DeFiLlama | `tvl < $10,000,000` | `tvl < $50,000,000` | `BRIDGE_TVL_LOW` |
| 11 | Bridge recent exploits | Security databases | Any exploit in 90 days | Any exploit in 180 days | `BRIDGE_EXPLOIT_RECENT` |
| 12 | Bridge estimated time | Bridge API | `time > 30 min` | `time > 10 min` | `BRIDGE_TOO_SLOW` |
| 13 | Hedge availability | `market_get_instruments(instType: "SWAP")` | -- | No matching perp | `NO_HEDGE_AVAILABLE` |
| 14 | Position size | User input | -- | `size_usd < $50,000` | `POSITION_SIZE_SMALL` |
| 15 | Spread vs cost | Calculation | `spread < total_cost + 20 bps` | `profit_to_cost < 1.5` | `NOT_PROFITABLE` |
| 16 | Price anomaly | Cross-chain comparison | `spread > 5000 bps (50%)` | `spread > 2000 bps (20%)` | `PRICE_ANOMALY` |

### 10.2 Check Execution Flow

```
START
  |
  +- Check 1-2: Infrastructure   -- BLOCK? -> Output error, STOP
  |
  +- Check 3: Freshness          -- BLOCK? -> Refetch data, retry once -> BLOCK? -> STOP
  |
  +- Check 4-7: Token Security   -- BLOCK? -> Move pair to blocked list, CONTINUE to next pair
  |   (on BOTH chains)              WARN?  -> Attach warning labels, CONTINUE
  |
  +- Check 8-9: Liquidity        -- BLOCK? -> Move to blocked list, CONTINUE to next pair
  |   (on BOTH chains)              WARN?  -> Attach liquidity note, CONTINUE
  |
  +- Check 10-12: Bridge Safety  -- BLOCK? -> Try next bridge -> All blocked? -> BLOCK pair
  |                                  WARN?  -> Attach bridge warning, CONTINUE
  |
  +- Check 13: Hedge             -- WARN?  -> Label as UNHEDGED, add risk score +3
  |
  +- Check 14: Position Size     -- WARN?  -> Show capital warning, CONTINUE
  |
  +- Check 15-16: Profitability  -- BLOCK? -> Exclude from results, CONTINUE to next pair
  |                                  WARN?  -> Include with margin warning
  |
  +- ALL PASSED -> Include in ranked results with accumulated WARNs
```

### 10.3 cross-chain-arbitrage Risk Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $100,000 | $500,000 | Maximum USD value per cross-chain arb trade |
| min_trade_size | $50,000 | $10,000 | Minimum recommended (can override to hard cap) |
| max_concurrent_arbs | 2 | 3 | Maximum simultaneous open cross-chain positions |
| min_net_profit | $100 | -- | Minimum net profit after all costs to recommend |
| min_net_profit_pct | 0.2% | -- | Minimum net profit as % of capital |
| max_price_age_sec | 10 | 10 | Maximum age of price data (more lenient than single-chain) |
| min_liquidity_usd | $500,000 | -- | Minimum DEX liquidity on each chain |
| max_slippage_pct | 0.5% | 1.0% | Maximum acceptable slippage per leg |
| max_bridge_time_min | 10 | 30 | Maximum bridge transit time for arb |
| min_bridge_tvl | $10,000,000 | -- | Minimum bridge pool TVL |
| max_hedge_leverage | 3x | 5x | Maximum hedge position leverage |

---

## 11. Fee Schedule & Cost Reference (Inlined)

### 11.1 OKX Perpetual Swap Trading Fees (for Hedge)

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.040% |
| VIP2 | >= 10M | 0.010% | 0.035% |
| VIP3 | >= 20M | 0.008% | 0.030% |
| VIP4 | >= 100M | 0.006% | 0.025% |
| VIP5 | >= 200M | 0.004% | 0.020% |

**Quick Reference: Round-Trip Hedge Taker Fees (open + close)**

| Tier | RT Fee | Fee on $50K | Fee on $100K |
|------|--------|-------------|-------------|
| VIP0 | 10.0 bps | $50.00 | $100.00 |
| VIP1 | 8.0 bps | $40.00 | $80.00 |
| VIP2 | 7.0 bps | $35.00 | $70.00 |
| VIP3 | 6.0 bps | $30.00 | $60.00 |
| VIP4 | 5.0 bps | $25.00 | $50.00 |
| VIP5 | 4.0 bps | $20.00 | $40.00 |

### 11.2 Bridge Cost Reference

| Bridge | Speed | Fee (stables) | Fee (ETH) | Min Transfer | Max Transfer | TVL Requirement |
|--------|-------|---------------|-----------|-------------|-------------|-----------------|
| Across | 1-4 min | 0.001% | 0.01% | $100 | $500K+ | > $10M |
| Stargate V2 | 2-5 min | 0.06% | 0.06% | $10 | $100K+ | > $10M |
| Synapse | 2-3 min | 0.04% | 0.04% | $10 | $100K+ | > $10M |

**Bridge Fee at $50,000 Trade Size:**

| Bridge | Stables Fee | ETH Fee |
|--------|------------|---------|
| Across | $0.50 | $5.00 |
| Stargate V2 | $30.00 | $30.00 |
| Synapse | $20.00 | $20.00 |

**NEVER USE for Arbitrage:**

| Bridge | Reason |
|--------|--------|
| Official L2 bridges (OP/Arb/Base native) | 7-day optimistic rollup withdrawal period |
| LayerZero (V1) | Fee > 1% in many cases |
| Any bridge with > 30 min transit | Spread will close before arrival |
| Any bridge with TVL < $10M | Liquidity risk too high for $50K+ |

### 11.3 Gas Costs per Chain

| Chain | chainIndex | Avg Gas Price | Swap Gas Limit | Approx Swap Cost | Native Token |
|-------|-----------|--------------|----------------|-------------------|-------------|
| Ethereum | 1 | 20-50 gwei | 150,000-300,000 | $5.00-$15.00 | ETH |
| Arbitrum | 42161 | 0.1-0.5 gwei | 1,000,000-2,000,000 | $0.10-$0.50 | ETH |
| Base | 8453 | 0.005-0.02 gwei | 150,000-300,000 | $0.01-$0.05 | ETH |
| Optimism | 10 | 0.005-0.02 gwei | 150,000-300,000 | $0.05-$0.20 | ETH |
| Polygon | 137 | 30-100 gwei | 150,000-300,000 | $0.01-$0.05 | MATIC |
| BSC | 56 | 3-5 gwei | 150,000-300,000 | $0.10-$0.30 | BNB |
| Solana | 501 | N/A (lamports) | N/A (compute units) | $0.001-$0.01 | SOL |

### 11.4 Chain Pair Opportunity Reference

| Pair | Avg Spread | Frequency | Min Capital | Notes |
|------|-----------|-----------|-------------|-------|
| Ethereum <-> Arbitrum | 0.5-1.5% | Daily | $50,000 | Most liquid; best for ETH/WBTC |
| Ethereum <-> Base | 0.5-2.0% | Daily | $50,000 | Growing liquidity; good for ETH |
| Stables across Polygon | 0.1-0.3% | Hourly | $100,000 | Thin margin; needs high capital |
| Ethereum <-> Optimism | 0.3-1.0% | Daily | $50,000 | Moderate liquidity |
| Altcoins across L2s | 2-10% | Random | $10,000 | Wide spread but high slippage risk |
| Arbitrum <-> Base | 0.3-0.8% | Daily | $50,000 | L2-to-L2; lowest gas costs |
| BSC <-> Ethereum | 0.5-2.0% | Weekly | $50,000 | Different ecosystems; less correlated |

### 11.5 Total Cost Template by Route

**Pattern A: Ethereum -> Arbitrum (ETH, $50K, Across)**
```
Gas source (Ethereum swap):      $15.00
Bridge fee (Across, 0.01%):       $5.00
Gas dest (Arbitrum swap):          $0.35
Slippage source (10 bps):        $50.00
Slippage dest (30 bps):         $150.00
Hedge open+close (VIP0):         $50.00
Hedge funding (~3 min):           $0.05
                                 ------
TOTAL:                          $270.40  (54.1 bps)
Breakeven spread:                54.1 bps
Min profitable spread (+20 bps): 74.1 bps
```

**Pattern B: Arbitrum -> Base (ETH, $50K, Across)**
```
Gas source (Arbitrum swap):        $0.35
Bridge fee (Across, 0.01%):       $5.00
Gas dest (Base swap):              $0.03
Slippage source (20 bps):       $100.00
Slippage dest (30 bps):         $150.00
Hedge open+close (VIP0):         $50.00
Hedge funding (~2 min):           $0.02
                                 ------
TOTAL:                          $305.40  (61.1 bps)
Breakeven spread:                61.1 bps
Min profitable spread (+20 bps): 81.1 bps
```

**Pattern C: Ethereum -> Base (USDC stables, $100K, Across)**
```
Gas source (Ethereum swap):      $10.00
Bridge fee (Across, 0.001%):      $1.00
Gas dest (Base swap):              $0.02
Slippage source (2 bps):         $20.00
Slippage dest (5 bps):           $50.00
Hedge: N/A (stables, no hedge):   $0.00
                                 ------
TOTAL:                           $81.02  (8.1 bps)
Breakeven spread:                 8.1 bps
Min profitable spread (+5 bps):  13.1 bps
```

---

## 12. Account Safety Protocol

### Demo vs Live Mode

The system defaults to **demo mode** at all times. Switching to live mode requires explicit user action.

**Demo Mode (Default):**
- MCP server config: `okx-DEMO-simulated-trading`
- All prices and positions are simulated
- All outputs include header: `[DEMO]`
- **No confirmation required to use demo mode**

**Live Mode:**
- MCP server config: `okx-LIVE-real-money`
- Real market data and real account positions
- All outputs include header: `[LIVE]`
- Recommendations are still analysis-only (no auto-execution)

**Switching from Demo to Live:**
```
1. User explicitly says "live", "真實帳戶", "real account", or similar
2. System confirms the switch with a clear warning:
   您正在切換至真實帳戶模式。
   - 所有數據將來自您的真實 OKX 帳戶
   - 建議仍為分析建議，不會自動執行交易
   - 跨鏈套利涉及多步驟操作，請格外謹慎
   - 最低建議資本: $50,000
   請確認：輸入 '確認' 或 'confirm' 繼續
3. User must reply with explicit confirmation
4. System verifies authentication via system_get_capabilities
5. If authenticated: switch and display [LIVE] header
6. If NOT authenticated: show AUTH_FAILED error, remain in demo
```

**Session Rules:**

| Rule | Description |
|------|-------------|
| Default on startup | Always demo mode |
| Timeout | If no activity for 30 minutes, revert to demo mode |
| Error fallback | If live mode encounters AUTH_FAILED, revert to demo with notification |
| Header requirement | EVERY output must show `[DEMO]` or `[LIVE]` -- no exceptions |
| No auto-execution | Even in live mode, skills only provide recommendations. `[RECOMMENDATION ONLY]` header is always present. |
| Capital warning | If size_usd < $50,000, always display `[MIN CAPITAL: $50,000]` warning |

---

## 13. Output Format & Templates (Inlined)

### 13.1 Formatting Rules

**Monetary Values:** `$12,345.67`, `+$1,234.56`, `-$89.10` (2 decimal places, comma thousands)
**Percentages:** 1 decimal place (e.g., `12.5%`). Net profit %: 2 decimal places.
**Basis Points:** Integer only (e.g., `99 bps`, `145 bps`)
**Risk Levels:** `[SAFE]`, `[WARN]`, `[BLOCK]`
**Timestamps:** `YYYY-MM-DD HH:MM UTC` (always UTC)
**Chain names:** Always in English

**Risk Gauge (Visual):**
```
1/10:  ▓░░░░░░░░░      LOW RISK
2/10:  ▓▓░░░░░░░░      LOW RISK
3/10:  ▓▓▓░░░░░░░      MODERATE-LOW
4/10:  ▓▓▓▓░░░░░░      MODERATE-LOW
5/10:  ▓▓▓▓▓░░░░░      MODERATE
6/10:  ▓▓▓▓▓▓░░░░      MODERATE
7/10:  ▓▓▓▓▓▓▓░░░      ELEVATED
8/10:  ▓▓▓▓▓▓▓▓░░      ELEVATED
9/10:  ▓▓▓▓▓▓▓▓▓░      HIGH RISK
10/10: ▓▓▓▓▓▓▓▓▓▓      HIGH RISK
```

**Sparklines:** Characters: `▁▂▃▄▅▆▇█` (8-24 data points)

**Section Separators:**
- Major sections: `══════════════════════════════════════════`
- Sub-sections: `── Section Name ──────────────────────────`
- Minor: `──────────────────────────────────────────`

### 13.2 Global Header Template

```
══════════════════════════════════════════
  Cross-Chain Arbitrage — {COMMAND}
  [{MODE}] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: OKX CEX + OKX DEX ({CHAINS})
══════════════════════════════════════════
```

### 13.3 Opportunity Table Template

```
── Opportunities Found: {COUNT} ────────────────────

  #{RANK}  {ASSET} ({SOURCE_CHAIN} -> {DEST_CHAIN})
  +- Spread:        {SPREAD_BPS} bps ({SPREAD_PCT}%)
  +- Direction:     {DIRECTION}
  +- Bridge:        {BRIDGE_NAME} (~{BRIDGE_TIME} min)
  +- Est. Profit:   {NET_PROFIT}  ({NET_PROFIT_PCT}%)
  +- Total Costs:   {TOTAL_COST}
  +- Profit/Cost:   {PROFIT_TO_COST}x
  +- Hedge:         {HEDGE_STATUS}
  +- Confidence:    {CONFIDENCE}
  +- Risk:          {RISK_LEVEL}

──────────────────────────────────────────
  Showing top {COUNT} of {TOTAL_SCANNED} pairs scanned
```

### 13.4 Cost Breakdown Template

```
── Cost Breakdown ──────────────────────────

  Trade Size:    {TRADE_SIZE}
  VIP Tier:      {VIP_TIER}

  Source Chain — {SOURCE_CHAIN} (Buy)
  +- Gas Cost:       {SOURCE_GAS}
  +- Slippage Est:   {SOURCE_SLIP}    ({SOURCE_SLIP_BPS} bps)
  +- Subtotal:       {SOURCE_TOTAL}

  Bridge — {BRIDGE_NAME}
  +- Bridge Fee:     {BRIDGE_FEE}     ({BRIDGE_FEE_PCT}%)
  +- Transit Time:   ~{BRIDGE_TIME} min

  Destination Chain — {DEST_CHAIN} (Sell)
  +- Gas Cost:       {DEST_GAS}
  +- Slippage Est:   {DEST_SLIP}      ({DEST_SLIP_BPS} bps)
  +- Subtotal:       {DEST_TOTAL}

  CEX Hedge — OKX {INST_ID}
  +- Open Short:     {HEDGE_OPEN}     (taker {HEDGE_FEE_BPS} bps)
  +- Close Short:    {HEDGE_CLOSE}    (taker {HEDGE_FEE_BPS} bps)
  +- Funding (~{BRIDGE_TIME} min): {HEDGE_FUNDING}
  +- Subtotal:       {HEDGE_TOTAL}

  ──────────────────────────────────────
  Gross Spread:      {GROSS_SPREAD}   ({GROSS_BPS} bps)
  Total Costs:      -{TOTAL_COST}     ({COST_BPS} bps)
  ══════════════════════════════════════
  Net Profit:        {NET_PROFIT}     ({NET_BPS} bps / {NET_PCT}%)
  Profit/Cost:       {PROFIT_TO_COST}x
  ══════════════════════════════════════
```

### 13.5 Execution Sequence Timeline Template

```
── Execution Sequence (Manual) ──────────────

  T+0:       Buy {ASSET} on {BUY_CHAIN} DEX
             Size: {SIZE_TOKENS} {ASSET} (~{SIZE_USD})
             Expected price: ~{BUY_PRICE}

  T+0:       Open SHORT hedge on OKX
             Instrument: {INST_ID}
             Size: {HEDGE_SIZE} contracts
             Leverage: {LEVERAGE}x

  T+1 min:   Initiate bridge transfer
             Bridge: {BRIDGE_NAME}
             Route: {SOURCE_CHAIN} -> {DEST_CHAIN}
             Amount: {BRIDGE_AMOUNT} {ASSET}

  T+{BRIDGE_END} min:  Bridge completes, tokens arrive on {DEST_CHAIN}
             VERIFY: Check wallet balance on {DEST_CHAIN}

  T+{SELL_START} min:  Sell {ASSET} on {SELL_CHAIN} DEX
             Size: {SIZE_TOKENS} {ASSET}
             Expected price: ~{SELL_PRICE}

  T+{HEDGE_CLOSE} min: Close SHORT hedge on OKX
             Close {HEDGE_SIZE} contracts

  T+{FINAL} min:      Calculate actual P&L
             Compare actual vs estimated

  ── IMPORTANT ────────────────────────────
  This is the recommended execution order.
  Each step must be performed MANUALLY.
  Do NOT proceed to the next step until
  the previous step is confirmed complete.
  如果橋接延遲超過 10 分鐘，評估是否繼續持有對沖。
══════════════════════════════════════════
```

### 13.6 Risk Gauge Template

```
── Risk Assessment ─────────────────────────

  Overall Risk:  {RISK_GAUGE}  {RISK_SCORE}/10
                 {RISK_LABEL}

  Breakdown:
  +- Execution Risk:   {EXEC_GAUGE}  {EXEC_SCORE}/10
  |                    {EXEC_NOTE}
  +- Market Risk:      {MKT_GAUGE}   {MKT_SCORE}/10
  |                    {MKT_NOTE}
  +- Contract Risk:    {SC_GAUGE}    {SC_SCORE}/10
  |                    {SC_NOTE}
  +- Bridge Risk:      {BR_GAUGE}    {BR_SCORE}/10
  |                    {BR_NOTE}
  +- Liquidity Risk:   {LIQ_GAUGE}   {LIQ_SCORE}/10
  |                    {LIQ_NOTE}
  +- Hedge Risk:       {HDG_GAUGE}   {HDG_SCORE}/10
                       {HDG_NOTE}
```

### 13.7 Next Steps Template

```
══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  {STEP_1}
  {STEP_2}
  {STEP_3}

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Cross-chain arbitrage involves bridge risk, smart contract risk,
  and multi-step execution. Past spreads do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  跨鏈套利涉及橋接風險、智能合約風險及多步驟執行。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 14. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp server unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify `~/.okx/config.toml`, restart server |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗。請檢查 OKX API 金鑰設定。 | API authentication failed. Check OKX API key config. | Update config.toml |
| `DATA_STALE` | Price data > 10s old (after retry) | 市場數據已過期（{venue} 延遲 {age}ms，跨鏈模式上限 10000ms）。 | Market data stale ({venue}: {age}ms, cross-chain mode max 10000ms). | Auto-retry once, then fail |
| `SECURITY_BLOCKED_SOURCE` | GoPlus check failed on source chain | 源鏈安全檢查未通過：{reason}。{asset} 在 {chain} 存在風險。 | Source chain security failed: {reason}. {asset} on {chain} is risky. | Show GoPlus findings, do not recommend |
| `SECURITY_BLOCKED_DEST` | GoPlus check failed on dest chain | 目標鏈安全檢查未通過：{reason}。{asset} 在 {chain} 存在風險。 | Dest chain security failed: {reason}. {asset} on {chain} is risky. | Show GoPlus findings, do not recommend |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用。所有結果標記為 [SECURITY UNCHECKED]。 | Security service unavailable. Results marked [SECURITY UNCHECKED]. | Retry once, then continue with warning labels |
| `INSUFFICIENT_LIQUIDITY_SOURCE` | DEX liquidity < $500K on source chain | {asset} 在 {source_chain} 的 DEX 流動性不足（${actual}，跨鏈套利最低要求 $500K）。 | {asset} DEX liquidity on {source_chain} insufficient (${actual}, min $500K). | Suggest different source chain |
| `INSUFFICIENT_LIQUIDITY_DEST` | DEX liquidity < $500K on dest chain | {asset} 在 {dest_chain} 的 DEX 流動性不足（${actual}，跨鏈套利最低要求 $500K）。 | {asset} DEX liquidity on {dest_chain} insufficient (${actual}, min $500K). | Suggest different dest chain |
| `BRIDGE_TVL_LOW` | Bridge pool TVL < $10M | {bridge} 橋接池 TVL 僅 ${tvl}，低於 $10M 最低要求。流動性風險過高。 | {bridge} pool TVL ${tvl} below $10M minimum. Liquidity risk too high. | Suggest alternative bridge |
| `BRIDGE_EXPLOIT_RECENT` | Bridge had security incident in 90 days | {bridge} 在 {date} 發生安全事件。建議 90 天內避免使用。 | {bridge} had a security incident on {date}. Avoid for 90 days. | Suggest alternative bridge |
| `BRIDGE_TOO_SLOW` | Bridge transit > 30 min | {bridge} 預估時間 {time} 分鐘，超過套利上限 30 分鐘。價差將在到達前消失。 | {bridge} estimated {time} min exceeds 30 min arb limit. Spread will close. | Suggest faster bridge or different route |
| `NO_HEDGE_AVAILABLE` | No OKX perp for the asset | {asset} 無 OKX 永續合約可供對沖。橋接期間價格風險無法消除。 | No OKX perp for {asset}. Price risk during bridge transit cannot be hedged. | Proceed unhedged (higher risk) or skip |
| `POSITION_SIZE_SMALL` | Size < $50K | 交易規模 ${size} 低於跨鏈套利建議最低 $50,000。固定成本可能超過利潤。 | Trade size ${size} below recommended $50K minimum. Fixed costs may exceed profit. | Show cost breakdown, explain why $50K+ needed |
| `NOT_PROFITABLE` | Net profit <= 0 after all costs | 扣除所有成本後淨利潤為負（{net_pnl}）。價差不足以覆蓋跨鏈成本。 | Net profit negative after all costs ({net_pnl}). Spread insufficient for cross-chain costs. | Show cost breakdown, suggest waiting for wider spread |
| `PRICE_ANOMALY` | Cross-chain spread > 50% | {asset} 的跨鏈價格差異異常大（{spread}%），可能是地址解析錯誤或死池。 | Price anomaly: {asset} cross-chain spread is {spread}%. Likely address error or dead pool. | Verify token addresses on both chains |
| `TOKEN_NOT_FOUND` | Token not found on target chain | 在 {chain} 上找不到 {asset} 的代幣合約。 | Token {asset} not found on {chain}. | Try alternative chains |
| `INSTRUMENT_NOT_FOUND` | OKX SWAP instrument missing | OKX 上找不到 {instId} 永續合約。 | Instrument {instId} not found on OKX. | Suggest similar instruments or unhedged approach |
| `MARGIN_TOO_THIN` | Profit-to-cost < 1.5 | 利潤空間偏薄（利潤/成本比 = {ratio}x）。跨鏈執行風險較高，建議等待更大價差。 | Thin margin (profit/cost = {ratio}x). Cross-chain execution risk elevated. Wait for wider spread. | Show sensitivity analysis |
| `RATE_LIMITED` | API rate limit hit after 3 retries | API 請求頻率超限，{wait} 秒後重試。 | API rate limit reached. Retrying in {wait}s. | Exponential backoff: 1s, 2s, 4s |
| `TRADE_SIZE_EXCEEDED` | Size > $500,000 hard cap | 交易金額 ${amount} 超過跨鏈套利上限 $500,000。 | Trade size ${amount} exceeds cross-chain arb limit $500,000. | Cap at $500,000 |
| `HEDGE_LEVERAGE_EXCEEDED` | Leverage > 3x (or 5x hard cap) | 對沖槓桿 {requested}x 超過上限 {limit}x。 | Hedge leverage {requested}x exceeds limit {limit}x. | Cap at limit |

---

## 15. Scan Results -- Complete Example

```
══════════════════════════════════════════
  Cross-Chain Arbitrage — SCAN
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX CEX + OKX DEX (Ethereum, Arbitrum, Base)
══════════════════════════════════════════

── 掃描參數 ──────────────────────────────

  資產:           ETH
  源鏈:           ethereum
  目標鏈:         arbitrum, base
  最低價差:       100 bps
  分析規模:       $50,000.00
  VIP 等級:       VIP0
  橋接偏好:       auto

── Opportunities Found: 2 ────────────────

  #1  ETH (Ethereum -> Arbitrum)
  +- Spread:        132 bps (1.32%)
  +- Direction:     Buy Ethereum DEX / Bridge / Sell Arbitrum DEX
  +- ETH Price:     $3,412.50 (Ethereum) / $3,457.54 (Arbitrum)
  +- Bridge:        Across (~2 min)
  +- Est. Profit:   +$383.20  (0.77%)
  +- Total Costs:   $276.80
  +- Profit/Cost:   1.38x
  +- Hedge:         ETH-USDT-SWAP SHORT (fully hedged)
  +- Confidence:    HIGH
  +- Risk:          [SAFE]

  #2  ETH (Ethereum -> Base)
  +- Spread:        108 bps (1.08%)
  +- Direction:     Buy Ethereum DEX / Bridge / Sell Base DEX
  +- ETH Price:     $3,412.50 (Ethereum) / $3,449.35 (Base)
  +- Bridge:        Across (~3 min)
  +- Est. Profit:   +$203.60  (0.41%)
  +- Total Costs:   $336.40
  +- Profit/Cost:   0.61x
  +- Hedge:         ETH-USDT-SWAP SHORT (fully hedged)
  +- Confidence:    MEDIUM
  +- Risk:          [WARN] 利潤空間偏薄 (profit/cost < 1.0x)

──────────────────────────────────────────
  Scanned: 2 chain pairs | Found: 2 opportunities

── Blocked Pairs ────────────────────────

  (none)

── Risk Assessment (Top Opportunity) ─────

  Overall Risk:  ▓▓▓▓░░░░░░  4/10
                 MODERATE-LOW

  Breakdown:
  +- Execution Risk:   ▓▓▓▓░░░░░░  4/10
  |                    Cross-chain, 5-step sequence
  +- Market Risk:      ▓▓░░░░░░░░  2/10
  |                    Fully hedged via OKX perp
  +- Contract Risk:    ▓░░░░░░░░░  1/10
  |                    Native ETH — no contract risk
  +- Bridge Risk:      ▓▓░░░░░░░░  2/10
  |                    Across: battle-tested, high TVL
  +- Liquidity Risk:   ▓▓░░░░░░░░  2/10
  |                    Deep pools on both chains
  +- Hedge Risk:       ▓░░░░░░░░░  1/10
                       ETH-USDT-SWAP: deep orderbook

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 詳細評估最佳機會:
     cross-chain-arbitrage evaluate --asset ETH --source-chain ethereum --dest-chain arbitrum --size-usd 50000
  2. 比較橋接選項:
     cross-chain-arbitrage bridge-compare --asset ETH --source-chain ethereum --dest-chain arbitrum --size-usd 50000
  3. 設計對沖倉位:
     cross-chain-arbitrage transit-hedge --asset ETH --size-usd 50000 --bridge-time-min 3

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Cross-chain arbitrage involves bridge risk, smart contract risk,
  and multi-step execution. Past spreads do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  跨鏈套利涉及橋接風險、智能合約風險及多步驟執行。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 15.2 Evaluate Results -- Complete Example

```
══════════════════════════════════════════
  Cross-Chain Arbitrage — EVALUATE
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:35 UTC
  Data sources: OKX CEX + OKX DEX (Ethereum, Arbitrum) + GoPlus + Across
══════════════════════════════════════════

── ETH 跨鏈套利評估 ─────────────────────

  Trade Size:       $50,000.00
  VIP Tier:         VIP0
  Route:            Ethereum -> Arbitrum (via Across)

── 價格比較 ──────────────────────────────

  Ethereum DEX
  +- Price:    $3,412.50
  +- Impact:   0.10% (at $50K swap quote)
  +- Data Age: 3s [SAFE]

  Arbitrum DEX
  +- Price:    $3,457.54
  +- Impact:   0.30% (at $50K swap quote)
  +- Data Age: 4s [SAFE]

  OKX CEX (Hedge Reference)
  +- Last:     $3,434.80
  +- Bid:      $3,434.50
  +- Ask:      $3,435.10
  +- Data Age: 1s [SAFE]

── 價差分析 ──────────────────────────────

  Gross Spread:     132 bps (1.32%)
  Spread USD:       $660.00
  Direction:        Buy Ethereum DEX / Bridge / Sell Arbitrum DEX
  Buy at:           Ethereum DEX @ ~$3,412.50
  Sell at:          Arbitrum DEX @ ~$3,457.54

── Safety Checks ───────────────────────────

  [  SAFE  ] Token Security (Ethereum)
              Native ETH — no contract risk

  [  SAFE  ] Token Security (Arbitrum)
              Native ETH — no contract risk

  [  SAFE  ] Liquidity (Ethereum)
              $892M DEX liquidity

  [  SAFE  ] Liquidity (Arbitrum)
              $245M DEX liquidity

  [  SAFE  ] Bridge Safety (Across)
              TVL: $385M | No exploits in 90 days | Speed: ~2 min

  [  SAFE  ] Hedge Available
              ETH-USDT-SWAP: deep orderbook ($12.5M depth at 10 bps)

  [  SAFE  ] Price Freshness
              All data < 5 seconds old

  ──────────────────────────────────────
  Overall: SAFE

── Bridge Selection ────────────────────────

  Selected: Across
  +- Speed:      1-4 min (est. ~2 min)
  +- Fee:        0.01% = $5.00
  +- TVL:        $385M [SAFE]
  +- Exploits:   None in 90 days

  Alternatives:
  +- Stargate V2: 2-5 min, 0.06% = $30.00  (+$25.00 vs Across)
  +- Synapse:     2-3 min, 0.04% = $20.00  (+$15.00 vs Across)

── Transit Hedge Design ────────────────────

  Instrument:      ETH-USDT-SWAP
  Side:            SHORT
  Size:            14.65 ETH (~$50,000)
  Leverage:        1x
  Margin Required: $50,000
  Duration:        ~2-4 min (until bridge completes + sell)
  Funding Cost:    $0.03 (negligible)

── Cost Breakdown ──────────────────────────

  Source Chain — Ethereum (Buy)
  +- Gas Cost:       $15.00
  +- Slippage Est:   $50.00     (10 bps)
  +- Subtotal:       $65.00

  Bridge — Across
  +- Bridge Fee:     $5.00      (0.01%)
  +- Transit Time:   ~2 min

  Destination Chain — Arbitrum (Sell)
  +- Gas Cost:       $0.35
  +- Slippage Est:   $150.00    (30 bps)
  +- Subtotal:       $150.35

  CEX Hedge — OKX ETH-USDT-SWAP
  +- Open Short:     $25.00     (taker 5.0 bps)
  +- Close Short:    $25.00     (taker 5.0 bps)
  +- Funding (~2 min): $0.03
  +- Subtotal:       $50.03

  ──────────────────────────────────────
  Gross Spread:      +$660.00   (132 bps)
  Total Costs:       -$270.38   (54.1 bps)
  ══════════════════════════════════════
  Net Profit:        +$389.62   (77.9 bps / 0.78%)
  Profit/Cost:       1.44x
  ══════════════════════════════════════

── Execution Sequence (Manual) ──────────────

  T+0:       Buy ETH on Ethereum DEX
             Size: 14.65 ETH (~$50,000)
             Expected price: ~$3,412.50

  T+0:       Open SHORT hedge on OKX
             Instrument: ETH-USDT-SWAP
             Size: 14.65 contracts
             Leverage: 1x

  T+1 min:   Initiate bridge transfer
             Bridge: Across
             Route: Ethereum -> Arbitrum
             Amount: 14.65 ETH

  T+3 min:   Bridge completes, ETH arrives on Arbitrum
             VERIFY: Check wallet balance on Arbitrum

  T+4 min:   Sell ETH on Arbitrum DEX
             Size: 14.65 ETH
             Expected price: ~$3,457.54

  T+5 min:   Close SHORT hedge on OKX
             Close 14.65 contracts

  T+5 min:   Calculate actual P&L
             Target: +$389.62 (0.78%)

  ── IMPORTANT ────────────────────────────
  This is the recommended execution order.
  Each step must be performed MANUALLY.
  Do NOT proceed to the next step until
  the previous step is confirmed complete.
  如果橋接延遲超過 10 分鐘，評估是否繼續持有對沖。

── Risk Assessment ─────────────────────────

  Overall Risk:  ▓▓▓▓░░░░░░  4/10
                 MODERATE-LOW

  Breakdown:
  +- Execution Risk:   ▓▓▓▓░░░░░░  4/10
  |                    5-step sequence; 2 min bridge
  +- Market Risk:      ▓▓░░░░░░░░  2/10
  |                    Fully hedged via ETH-USDT-SWAP short
  +- Contract Risk:    ▓░░░░░░░░░  1/10
  |                    Native ETH on both chains
  +- Bridge Risk:      ▓▓░░░░░░░░  2/10
  |                    Across: $385M TVL, battle-tested
  +- Liquidity Risk:   ▓▓░░░░░░░░  2/10
  |                    $892M (ETH) + $245M (ARB) DEX liquidity
  +- Hedge Risk:       ▓░░░░░░░░░  1/10
                       Deep perp orderbook, $12.5M depth

══════════════════════════════════════════
  Recommended Action: PROCEED (with manual execution)
══════════════════════════════════════════

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 如滿意分析結果，按上方執行序列手動操作 5 步
  2. 考慮不同規模:
     cross-chain-arbitrage evaluate --asset ETH --source-chain ethereum --dest-chain arbitrum --size-usd 100000
  3. 設計對沖細節:
     cross-chain-arbitrage transit-hedge --asset ETH --size-usd 50000 --bridge-time-min 3

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Cross-chain arbitrage involves bridge risk, smart contract risk,
  and multi-step execution. Past spreads do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  跨鏈套利涉及橋接風險、智能合約風險及多步驟執行。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 16. Cross-Skill Integration Contracts

### Input: What This Skill Consumes

| Source Skill / Tool | Data Consumed | Schema | Usage |
|---------------------|--------------|--------|-------|
| OnchainOS `dex-market price` | DEX price on source chain | `{priceUsd}` | Source chain price in Step 3a |
| OnchainOS `dex-market price` | DEX price on dest chain | `{priceUsd}` | Dest chain price in Step 3a |
| `market_get_ticker` | CEX price for hedge reference | `{last, bidPx, askPx, ts}` | Hedge sizing in Step 3a |
| `market_get_orderbook` | CEX orderbook depth for hedge | `{asks, bids}` | Hedge slippage estimation |
| `market_get_instruments` | Available perpetual swaps | `{instId, ctVal, ctMult}` | Hedge availability check |
| `market_get_funding_rate` | Current funding rate | `{fundingRate, nextFundingRate}` | Hedge funding cost estimation |
| GoPlus `check_token_security` | Token security on source chain | GoPlus response object | Safety gate in Step 2b |
| GoPlus `check_token_security` | Token security on dest chain | GoPlus response object | Safety gate in Step 2b |
| OnchainOS `dex-token search` | Token address resolution | `{address, symbol, chain, decimals}` | Address resolution in Step 2a |
| OnchainOS `dex-token price-info` | DEX liquidity depth | `{liquidityUsd}` | Liquidity validation in Step 2d |
| OnchainOS `dex-swap quote` | Executable DEX price + impact | `{toTokenAmount, priceImpactPercent, estimatedGas}` | Slippage + gas estimation |

### Output: What This Skill Produces

| Output | Consumer | Schema | Handoff |
|--------|----------|--------|---------|
| Ranked opportunity list | User (formatted output) | `CrossChainScanResult` | Displayed directly |
| Single evaluation | User (formatted output) | `CrossChainEvaluateResult` | Displayed directly |
| Bridge comparison | User (formatted output) | `BridgeCompareResult` | Displayed directly |
| Transit hedge design | User (formatted output) | `TransitHedgeResult` | Displayed directly |

### Data Flow Diagram

```
OnchainOS dex-market price --chain {source} --token {addr}  (Source DEX price)
OnchainOS dex-market price --chain {dest} --token {addr}    (Dest DEX price)
market_get_ticker({ASSET}-USDT)                              (CEX hedge ref)
  |
  |  Cross-chain price snapshot
  v
cross-chain-arbitrage.scan / evaluate
  |
  +- GoPlus check_token_security(source_chain_id, addr)
  |    -> BLOCK / WARN / SAFE (source)
  |
  +- GoPlus check_token_security(dest_chain_id, addr)
  |    -> BLOCK / WARN / SAFE (dest)
  |
  +- Bridge validation (TVL, exploits, speed)
  |    -> BLOCK if unsafe / WARN if marginal
  |
  +- OnchainOS dex-token price-info -> liquidityUsd (both chains)
  |    -> BLOCK if < $500K on either chain
  |
  +- OnchainOS dex-swap quote (both chains) -> slippage + gas
  |
  +- market_get_instruments(SWAP) -> hedge availability
  +- market_get_orderbook -> hedge depth
  +- market_get_funding_rate -> hedge funding cost
  |
  |  All-in cost calculation
  v
cross-chain-arbitrage applies go/no-go:
  |  - net_profit > 0?
  |  - spread > total_cost + 20 bps?
  |  - security SAFE on both chains?
  |  - bridge TVL > $10M?
  v
Output: Ranked opportunities with safety checks + cost breakdowns + hedge design
```

---

## 17. Chain Reference

| Chain | chainIndex | Native Token | Native Address (EVM) | DEX Ecosystem |
|-------|-----------|-------------|---------------------|---------------|
| Ethereum | 1 | ETH | 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee | Uniswap V3, 1inch |
| BSC | 56 | BNB | 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee | PancakeSwap |
| Polygon | 137 | MATIC | 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee | Uniswap V3, QuickSwap |
| Optimism | 10 | ETH | 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee | Uniswap V3, Velodrome |
| Arbitrum | 42161 | ETH | 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee | Uniswap V3, Camelot |
| Base | 8453 | ETH | 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee | Uniswap V3, Aerodrome |
| Solana | 501 | SOL | native | Jupiter, Raydium, Orca |

**Common Bridgeable Token Addresses:**

| Token | Ethereum | Arbitrum | Base | Optimism | Polygon |
|-------|----------|----------|------|----------|---------|
| WETH | 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 | 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1 | 0x4200000000000000000000000000000000000006 | 0x4200000000000000000000000000000000000006 | 0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619 |
| USDC | 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 | 0xaf88d065e77c8cC2239327C5EDb3A432268e5831 | 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 | 0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85 | 0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359 |
| USDT | 0xdAC17F958D2ee523a2206206994597C13D831ec7 | 0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9 | -- | 0x94b008aA00579c1307B0EF2c499aD98a8ce58e58 | 0xc2132D05D31c914a87C6611C10748AEb04B58e8F |
| WBTC | 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 | 0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f | -- | 0x68f180fcCe6836688e9084f035309E29Bf0A2095 | 0x1BFD67037B42Cf73acF2047067bd4F2C47D9BfD6 |

---

## 18. Supported Bridge Routes

Not all bridge protocols support all chain pairs. Use this matrix to determine valid routes:

| Route | Across | Stargate V2 | Synapse |
|-------|--------|-------------|---------|
| Ethereum <-> Arbitrum | Yes | Yes | Yes |
| Ethereum <-> Base | Yes | Yes | No |
| Ethereum <-> Optimism | Yes | Yes | Yes |
| Ethereum <-> Polygon | Yes | Yes | Yes |
| Ethereum <-> BSC | No | Yes | Yes |
| Arbitrum <-> Base | Yes | Yes | No |
| Arbitrum <-> Optimism | Yes | Yes | Yes |
| Arbitrum <-> Polygon | Yes | Yes | No |
| Base <-> Optimism | Yes | Yes | No |

**Solana bridges:** Solana requires specialized bridges (Wormhole, deBridge). These are NOT recommended for arbitrage due to higher fees and slower transit. Cross-chain arb with Solana should be approached with extra caution.
